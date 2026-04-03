# Data Modeling

Covers OLTP vs OLAP design patterns, schema types, and modeling best practices.

---

## OLTP vs OLAP

| Feature | OLTP (RDS, DynamoDB) | OLAP (Redshift, Athena) |
|---------|---------------------|------------------------|
| **Focus** | Transactions, precise lookups | Analytics, aggregations |
| **Queries** | Simple, return few rows | Complex, huge scans |
| **Data** | Current, volatile | Historical, static |
| **Normalization** | 3NF (avoid redundancy) | Denormalized (Star/Snowflake) |
| **Writes** | Row-by-row | Batch loads |
| **Latency** | Low (ms) | Higher (seconds to minutes) |

---

## Schema Designs for OLAP

### Star Schema
- **Fact table** in the center (transactions, events, measures)
- **Dimension tables** around it (who, what, where, when)
- Simple joins (1 level)
- **Preferred for Redshift**
- Example: `fact_sales` joined to `dim_customer`, `dim_product`, `dim_date`

### Snowflake Schema
- Normalized dimension tables (dimensions have sub-dimensions)
- More complex joins (multiple levels)
- Less storage (reduced redundancy)
- Slower query performance than Star

### Star vs Snowflake for Exam
- Default answer is usually **Star Schema** for Redshift/data warehouse
- Snowflake Schema when storage optimization is priority over query speed

### Fact Table Types

Different business requirements call for different fact table designs:

| Type | Structure | Use Case | Example |
|---|---|---|---|
| **Transactional** | One row per event | Record individual business events | One row per order, payment, or login |
| **Periodic Snapshot** | One row per time period per entity | Measure state at regular intervals | Monthly account balance, weekly inventory level |
| **Accumulating Snapshot** | One row per process instance, updated as it progresses | Track a pipeline with known stages | One row per order, updated as it moves through picking → packed → shipped → delivered |

Exam pattern: "Track how long each order spends in each fulfillment stage" → Accumulating Snapshot (one row per order, timestamp columns per stage, updated at each transition).

### Kimball vs Inmon

Two competing data warehouse design philosophies:

| Aspect | Kimball (Bottom-Up) | Inmon (Top-Down) |
|---|---|---|
| **Approach** | Build data marts first, integrate via conformed dimensions | Build enterprise DW in 3NF first, derive data marts |
| **Schema** | Dimensional (Star/Snowflake) | Normalized (3NF) |
| **Speed to value** | Faster — deliver one data mart at a time | Slower — full EDW must be built first |
| **Query speed** | Faster (denormalized) | Slower (many joins) |
| **AWS / Redshift alignment** | Kimball (Star schema preferred) | Less common on AWS |

Exam pattern: "A company wants to build a data warehouse incrementally, delivering business value one domain at a time" → Kimball methodology (bottom-up, data marts with conformed dimensions).

---

## DynamoDB Data Modeling

### Single Table Design
- Store multiple entity types in one table
- Use Partition Key and Sort Key to model relationships
- Example: PK=`CUSTOMER#123`, SK=`ORDER#456`
- Reduces the need for joins (which DynamoDB doesn't support)

### Access Pattern First
- DynamoDB models are designed around **access patterns**, not relationships
- Define all queries before designing the table
- Use GSIs for additional access patterns

### Global Secondary Indexes (GSI) vs Local Secondary Indexes (LSI)

| Feature | GSI | LSI |
|---|---|---|
| **Creation** | Any time (before or after table creation) | Must be created at table creation time |
| **Partition Key** | Can use any attribute as PK | Must use same PK as base table |
| **Sort Key** | Any attribute | Different attribute from base table |
| **Consistency** | Eventually consistent only | Strongly consistent supported |
| **Capacity** | Has its own RCU/WCU | Shares table's RCU/WCU |
| **Limit** | 20 per table | 5 per table |

### Hot Partition Problem

**Symptom:** DynamoDB throttles writes/reads despite provisioned capacity appearing sufficient.

**Cause:** Many requests targeting the same partition key value route to a single partition, which has a hard limit of 3,000 RCU or 1,000 WCU. Even with 10,000 WCU provisioned across the table, a single hot partition is capped at 1,000 WCU.

**Common causes:**
- Using a date or status field as partition key (all today's writes hit the same partition)
- A "viral" item (one product ID getting all the traffic)

**Fix — Write Sharding:**
Append a random suffix to the partition key to spread load across multiple physical partitions:

```python
import random

# Instead of PK = "2024-01-15" (all writes go to one partition)
# Use a random shard suffix
shard_count = 10
shard = random.randint(0, shard_count - 1)
pk = f"2024-01-15#{shard}"  # e.g., "2024-01-15#7"

# Reading all shards requires scatter-gather (query each shard, merge results)
results = []
for shard in range(shard_count):
    response = table.query(KeyConditionExpression=Key('pk').eq(f'2024-01-15#{shard}'))
    results.extend(response['Items'])
```

**Exam patterns:**
- "DynamoDB is throttling despite sufficient total capacity — all traffic is to one item" → Hot partition; apply write sharding
- "DynamoDB table uses order_date as partition key — experiencing uneven performance" → Hot partition anti-pattern; redesign PK or add shard suffix

---

## Slowly Changing Dimensions (SCD)

Slowly Changing Dimensions (SCDs) describe how dimension table data changes over time. This is explicitly in the DEA-C01 exam guide. Three types appear in exam questions.

### SCD Type 1 — Overwrite (No History)

Overwrite the old value with the new value. No historical record is kept.

```sql
-- Customer moved from New York to Austin
UPDATE dim_customer
SET city = 'Austin', state = 'TX'
WHERE customer_id = 123;
-- History: New York is gone forever
```

Use when: History is irrelevant. The current value is all that matters.
Example: Correcting a data entry error (wrong phone number).

### SCD Type 2 — Add New Row (Full History) — Most Common in Data Warehouses

Each change creates a NEW row with a new surrogate key. The old row is marked inactive. The active row is identified by `is_current = TRUE` or by `effective_to = '9999-12-31'`.

```sql
-- dim_customer state after customer moved:
-- sk | customer_id | city     | state | effective_from | effective_to | is_current
-- 1  | 123         | New York | NY    | 2022-01-01     | 2024-03-14   | FALSE
-- 2  | 123         | Austin   | TX    | 2024-03-15     | 9999-12-31   | TRUE

-- Query current address:
SELECT city, state FROM dim_customer
WHERE customer_id = 123 AND is_current = TRUE;

-- Ad-hoc lookup: address as of 2023-06-01 (when you only have the natural key):
SELECT city, state FROM dim_customer
WHERE customer_id = 123
  AND effective_from <= '2023-06-01'
  AND effective_to > '2023-06-01';
```

**Critical implementation detail — fact tables store the surrogate key at write time:**
The fact table records `customer_sk` (the surrogate key) at the moment the transaction occurs. This permanently binds the fact row to the correct dimension version. Joins to `dim_customer` use the surrogate key directly — no date range logic needed at query time:

```sql
-- Standard fact-to-dimension join (no date range needed):
SELECT f.order_id, f.amount, c.city, c.state
FROM fact_orders f
JOIN dim_customer c ON f.customer_sk = c.sk;
-- c.sk = 1 → New York (for orders placed before the move)
-- c.sk = 2 → Austin (for orders placed after the move)
```

Date range queries are only needed for ad-hoc lookups when the surrogate key is not available.

AWS implementation: Redshift is designed for Type 2. Use surrogate keys (`BIGINT IDENTITY`) as the primary key on dimension tables.

### SCD Type 3 — Add New Column (Limited History)

Add a new column to store the previous value. Only one historical value is tracked.

```sql
-- Add previous_city column
ALTER TABLE dim_customer ADD COLUMN previous_city VARCHAR(100);

-- On change: shift current to previous, update current
UPDATE dim_customer
SET previous_city = city, city = 'Austin'
WHERE customer_id = 123;

-- Result:
-- customer_id | city   | previous_city
-- 123         | Austin | New York
-- (only remembers the immediate previous value — anything older is lost)
```

Use when: You only need to track the most recent change, not full history.
Example: "What was the customer's previous loyalty tier before the current one?"

### SCD Decision Guide

```
Do you need full history of all changes over time?
  YES → SCD Type 2 (new row per change, surrogate key, effective dates)

Do you need only the most recent previous value?
  YES → SCD Type 3 (previous_value column)

Is history irrelevant? (or correcting a data error)
  YES → SCD Type 1 (overwrite)
```

**Exam patterns:**
- "Track which city each customer lived in at the time of each order" → SCD Type 2 (fact table stores surrogate key at write time)
- "Customer changed loyalty tier — analysts need to know both current and previous tier only" → SCD Type 3
- "Correct a misspelled customer name — no history needed" → SCD Type 1

---

## Data Lineage and AWS Tooling

Data lineage tracks the origin, movement, and transformation of data throughout its lifecycle. It answers: "Where did this data come from? What transformations were applied? Who consumed it?"

**Why it matters for the exam:**
- Compliance requirements (GDPR, HIPAA) require proving data provenance
- Debugging: when a dashboard shows wrong numbers, lineage tells you which pipeline step introduced the error
- Impact analysis: if a source table schema changes, lineage shows which downstream pipelines and reports are affected

### Amazon SageMaker ML Lineage Tracking

SageMaker ML Lineage Tracking is in the official DEA-C01 in-scope services list. It tracks the relationships between:
- Datasets (input data)
- Training jobs (processing steps)
- Models (outputs)
- Endpoints (deployments)

Key concepts:
- **Lineage graph:** directed acyclic graph (DAG) connecting artifacts → executions → artifacts
- **Artifact:** any versioned data entity (dataset S3 URI, model artifact, etc.)
- **Association:** a directed link between two entities (e.g., dataset "was input to" training job)
- **Context:** a grouping of related lineage entities (e.g., all artifacts in one ML project)

Query lineage:
```python
import boto3
sm = boto3.client('sagemaker')

# Find all artifacts downstream of a specific dataset
response = sm.list_associations(
    SourceArn='arn:aws:sagemaker:us-east-1:123:artifact/dataset-abc',
    AssociationType='ContributedTo'
)
# Returns: training jobs, models, and endpoints that used this dataset
```

**Exam pattern:** "Track which training dataset was used to produce a specific deployed model for compliance auditing" → SageMaker ML Lineage Tracking.

### AWS Glue Data Catalog for Lineage

Glue Data Catalog tracks schema evolution (version history of table schemas) which provides a basic form of structural lineage. When a Glue Crawler updates a table schema, the previous version is retained and queryable.

### Third-Party Lineage Tools on AWS

OpenLineage (open standard) and Apache Atlas are commonly integrated with AWS data pipelines for full operational lineage. Know that AWS-native lineage is primarily SageMaker-scoped (for ML workloads) — for general ETL lineage, third-party tools are typically used.

---

## Glue Schema Registry

AWS Glue Schema Registry is a centralized repository for managing schemas for data streams (Kinesis, MSK/Kafka) and other data sources.

**The Problem It Solves:**

Without schema registry: producers write records in any schema they choose. Consumers must handle schema inconsistencies, unexpected fields, and breaking changes manually.

With schema registry: producers register their schema before writing. The registry validates every record against the registered schema. Consumers know exactly what schema to expect.

**Compatibility Modes (exam-tested):**

| Mode | Rule | Upgrade Order |
|---|---|---|
| **BACKWARD** | New schema can read records written with old schema | Upgrade **consumers** first — they handle both old and new records |
| **FORWARD** | Old schema can read records written with new schema | Upgrade **producers** first — old consumers still process new records |
| **FULL** | Both backward and forward compatible | Either side can upgrade first — safest for production |
| **NONE** | No compatibility check | No safety — any schema accepted |

**Schema change compatibility reference:**

| Change | BACKWARD | FORWARD | FULL |
|---|---|---|---|
| Add optional field with a default value | ✓ (new readers use default for old records) | ✓ (old readers ignore unknown field) | ✓ |
| Remove a field | ✓ (new schema ignores extra fields in old records) | ✗ (old readers expect field, not present in new records) | ✗ |
| Change a field's data type | ✗ | ✗ | ✗ |
| Add a required field (no default) | ✗ (old records missing required field) | ✓ | ✗ |

Integration with Kinesis:
```python
import boto3

schema_registry_client = boto3.client('glue')

schema_definition = '''
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "status", "type": "string"}
  ]
}
'''
# Register schema in registry (only needs to happen once per schema version)
response = schema_registry_client.register_schema_version(
    SchemaId={'RegistryName': 'prod-registry', 'SchemaName': 'OrderEvent'},
    SchemaDefinition=schema_definition
)
```

**Exam patterns:**
- "How do you ensure a schema change to a Kinesis stream doesn't break downstream consumers?" → Glue Schema Registry with BACKWARD compatibility
- "Producer added a new optional field to Kafka messages — old consumers must still work" → BACKWARD or FULL compatibility (adding optional field with default is FULL compatible)
- "Centrally enforce that all Kinesis producers use an approved schema" → Glue Schema Registry with FULL compatibility

---

## Redshift Schema Design

### Distribution Styles

Redshift distributes table rows across compute nodes. Choosing the right style avoids data movement during joins (the main cause of slow queries).

| Style | How Rows Are Distributed | Best For |
|---|---|---|
| **KEY** | Rows with same key value go to same node | Large tables frequently joined — co-locates matching rows |
| **ALL** | Full copy of table on every node | Small dimension tables — eliminates broadcast joins |
| **EVEN** | Round-robin across all nodes | Tables with no good join key or rarely joined |
| **AUTO** | Redshift chooses (ALL for small, EVEN for larger) | Default for new tables — let Redshift decide |

**Distribution key selection rules:**
1. Identify the most common JOIN column on large fact tables → use as DISTKEY
2. Small dimension tables (< a few million rows) → ALL distribution
3. No natural join key → EVEN
4. New table with unknown query patterns → AUTO

### Sort Keys

Sort keys determine the physical sort order of rows on disk. Redshift skips entire blocks that don't match a WHERE clause filter (zone maps) — only effective when data is physically sorted by the filter column.

**Two sort key types:**

| Type | Behavior | Status |
|---|---|---|
| **Compound** | Sorts by columns in declared order (leftmost column most significant) | Active — use this |
| **Interleaved** | Treats each column with equal weight | **Deprecated** — do not create new tables with this |

Compound sort keys are the correct choice for all new tables. Interleaved sort keys have been **deprecated by AWS** — `VACUUM REINDEX` (required to maintain their effectiveness) is prohibitively slow on large tables. For unpredictable multi-column filter patterns, use a compound sort key on the most commonly filtered column — not interleaved.

> **Exam trap:** If a question asks which sort key handles unpredictable multi-column filters, the answer is still **compound sort key** — interleaved is a distractor.

### VACUUM and ANALYZE

**VACUUM:** Reclaims disk space from soft-deleted rows and re-sorts unsorted rows (inserts after initial load go to unsorted region).

| VACUUM Type | What It Does |
|---|---|
| `VACUUM FULL` | Re-sort + reclaim space (default) |
| `VACUUM SORT ONLY` | Re-sort only, no space reclaim |
| `VACUUM DELETE ONLY` | Reclaim space only, no re-sort |
| `VACUUM REINDEX` | Required for interleaved sort keys |

**ANALYZE:** Updates table statistics used by the query planner. Without current statistics, the planner may choose inefficient join orders or scan strategies.

```sql
-- Run after bulk loads or significant updates/deletes
VACUUM FULL my_fact_table;
ANALYZE my_fact_table;
```

**Exam pattern:** "Redshift queries are slow after a nightly bulk DELETE and re-load" → Run VACUUM to reclaim space from deleted rows and re-sort; run ANALYZE to refresh statistics.

---

## Data Mesh vs Data Lake vs Data Lakehouse

| Concept | Type | Key Characteristic |
|---------|------|--------------------|
| **Data Lake** | Technical architecture | Centralized S3 storage; raw + processed data; governed by central team |
| **Data Mesh** | Organizational pattern | Decentralized domain ownership; domain teams produce and own their data products |
| **Data Lakehouse** | Technical architecture | S3 storage + ACID transactions + data warehouse query capabilities via open table formats |

### Data Lakehouse

Combines the scale and cost of a data lake with the reliability and performance features of a data warehouse. Implemented via open table formats on S3:

| Format | AWS Services | Key Features |
|---|---|---|
| **Apache Iceberg** | S3 Tables, Athena, EMR, Glue | ACID, time travel, hidden partitioning, schema evolution |
| **Delta Lake** | EMR, Glue | ACID, time travel, schema enforcement |
| **Apache Hudi** | EMR, Glue | ACID, incremental processing, upserts/deletes on S3 |

**Why Data Lakehouse matters for exam:**
- Enables `UPDATE` and `DELETE` on data lake tables (not possible with plain S3 Parquet)
- Time travel: query data as of a past snapshot (`SELECT * FROM table FOR TIMESTAMP AS OF '2024-01-01'`)
- Upserts: merge CDC changes from DMS/DynamoDB Streams into a lake table without rewriting entire partitions

**Exam pattern:** "Need to apply GDPR right-to-erasure (delete specific rows) on a petabyte S3 data lake without rewriting all files" → Data Lakehouse with Apache Iceberg (row-level deletes via Iceberg's merge-on-read or copy-on-write).

### Data Mesh Key Properties
- **Domain ownership:** The team that produces data is responsible for its quality, availability, and documentation
- **Data as a product:** Domains publish well-defined data products with SLAs, not raw dumps
- **Self-serve platform:** Central platform team provides infrastructure (S3, Glue, Redshift) but not data ownership
- **Federated governance:** Standards (data quality, security) are set centrally but enforced locally

---

## Exam Gotchas

- **Star Schema** is the default answer for Redshift data modeling
- DynamoDB modeling is **access pattern first**, not relationship first
- **Data Mesh** is organizational (decentralized domains), **Data Lake** is technical (centralized S3)
- Know the difference between normalization (OLTP, 3NF) and denormalization (OLAP, Star)
- Materialized Views in Redshift pre-compute complex aggregations for faster queries
- **SCD Type 2 is the default answer for "track historical changes" in a data warehouse.** Fact tables store the surrogate key at write time — the join to dim_customer needs no date range logic. Date range queries are only for ad-hoc lookups without a surrogate key.
- **Glue Schema Registry: BACKWARD = consumers upgrade first (new schema reads old data). FORWARD = producers upgrade first (old schema reads new data). FULL = either side can upgrade first.** Adding an optional field with a default is FULL compatible (both directions work). Removing a field is only BACKWARD compatible (breaks old readers expecting the field).
- **SageMaker ML Lineage Tracking is specifically for ML workloads** — tracking datasets → training jobs → models → endpoints. It is NOT a general-purpose ETL lineage tool. For tracking which Glue job transformed which S3 file, SageMaker lineage is not the right tool; CloudWatch Logs + EventBridge records are more appropriate.
- **Snowflake schema is almost never the right answer for Redshift.** Star schema with denormalized dimensions is preferred for Redshift because it minimizes JOIN depth and data movement. Choose snowflake schema only if a question specifically prioritizes storage reduction over query speed.
- **DynamoDB hot partition = provisioned capacity is sufficient in total but throttling still occurs.** Fix: write sharding (append random suffix to PK). LSIs must be created at table creation time; GSIs can be added any time.
- **Redshift DELETE does not reclaim disk space** — deleted rows are soft-marked. Run `VACUUM` to reclaim space and `ANALYZE` to refresh statistics after bulk operations.
- **Accumulating Snapshot fact tables are updated in place** — unlike transactional facts (append-only), accumulating snapshots have one row per process instance that gets updated as the process moves through stages. Redshift handles this via `MERGE` or staged `UPDATE`.
- **Data Lakehouse (Iceberg/Delta/Hudi) is the answer when a question requires row-level deletes or upserts on an S3 data lake** — plain Parquet on S3 is immutable; only open table formats support row-level mutations without full partition rewrites.

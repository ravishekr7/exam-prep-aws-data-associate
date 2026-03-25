# AWS Glue

Serverless data integration service for discovering, preparing, and combining data.

---

## 1. Glue Data Catalog

A central metadata repository (Hive-compatible metastore). It does **not** store data, only metadata (table definitions, schema, partitions).

### Key Concepts
- **Databases:** Logical grouping of tables
- **Tables:** Metadata definitions (columns, types) pointing to data in S3, JDBC, etc.
- **Connections:** Store credentials and network info (VPC, Subnet, Security Groups) to connect to data sources (RDS, Redshift)

### Crawlers
Automatically scan data sources (S3, DynamoDB, JDBC) to infer schema and populate the Catalog.
- **Schedule:** On-demand, hourly, daily, etc.
- **Schema Change Behavior:** Update, Log, or Ignore
- **Partition Indexing:** Improves query performance for large tables in S3

### Exam Tip
Crawlers incur cost and take time. If schema is known/static, use **Partition Projection** or event-based table updates instead.

---

## 2. Glue ETL Jobs

### Job Types
| Type | Description | Use Case |
|------|-------------|----------|
| **Spark (Python/Scala)** | Distributed processing | Heavy workloads, complex transforms |
| **Python Shell** | Single node, lightweight | Small tasks (triggering SQL scripts). Cheaper than Spark |
| **Ray** | Python/Pandas across multiple nodes | Pandas-based distributed processing |
| **Streaming ETL** | Consumes from Kinesis/Kafka | Near real-time writes to S3/Redshift |

### Worker Types
| Type | Specs | Use Case |
|------|-------|----------|
| **G.1X** | 1 DPU (4 vCPU, 16GB) | Good balance for most jobs |
| **G.2X** | 2 DPU (8 vCPU, 32GB) | High memory needs (ML, large joins) |
| **Flex** | Standard workers on spare capacity | Cheaper but can be interrupted (like Spot) |

### DynamicFrame vs DataFrame
| Feature | DynamicFrame | DataFrame |
|---------|-------------|-----------|
| Schema | No enforced schema. Rows can differ | Enforced schema |
| Best for | Nested/messy data | Structured data |
| Features | `ApplyMapping` for schema conversion | Standard Spark operations |
| Convert | `.toDF()` to get DataFrame | `.fromDF()` to get DynamicFrame |

---

## 3. Glue Studio and DataBrew

- **Glue Studio:** Visual interface to build DAGs for ETL without writing code
- **Glue DataBrew:** No-code visual data preparation tool for analysts/data scientists to clean and normalize data

---

## 4. Glue Schema Registry

Centrally manage schemas for Kinesis/Kafka streams.
- Enforces compatibility modes: **Backward**, **Forward**, **Full**
- Prevents "bad" data from entering the stream
- Supports schema versioning and evolution

---

## 5. Glue Data Quality

Define rules using DQDL (Data Quality Definition Language):
```
Rules = [
  "IsComplete \"col_a\"",
  "ColumnValues \"age\" > 0",
  "Column \"email\" is_unique"
]
```

Actions on failure: Fail job, write to CloudWatch, route bad data to separate S3 bucket.

---

## 6. Glue Streaming ETL

Streaming ETL jobs differ from batch jobs in that they run continuously, processing data in **micro-batches** rather than processing a fixed dataset. The job never terminates on its own — it keeps consuming from the source indefinitely.

### Key Differences from Batch
- **Micro-batch model:** Glue Streaming ETL reads records in time windows (e.g., every 100 seconds), processes them, and writes to the destination.
- **Checkpoint location:** Glue stores its position in the stream to an S3 path. If the job restarts, it resumes from the last checkpoint, not from the beginning.
- **Always-on cost:** Unlike batch jobs, streaming jobs bill continuously (per DPU-hour) since they never terminate.

### Architecture Pattern
```
Kinesis Data Streams → Glue Streaming ETL → S3 / Redshift
```

### Code Snippet: Read from Kinesis in Glue Streaming Job
```python
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)

# Create a streaming DynamicFrame from Kinesis
kinesis_stream = glueContext.create_data_frame_from_options(
    connection_type="kinesis",
    connection_options={
        "typeOfData": "kinesis",
        "streamARN": "arn:aws:kinesis:us-east-1:123456789:stream/my-stream",
        "classification": "json",
        "startingPosition": "TRIM_HORIZON",
        "inferSchema": "true",
    },
    transformation_ctx="kinesis_stream"
)

# Process with a window function
def processBatch(data_frame, batchId):
    if data_frame.count() > 0:
        dynamic_frame = DynamicFrame.fromDF(data_frame, glueContext, "dynamic_frame")
        # Apply transformations...
        glueContext.write_dynamic_frame.from_options(
            frame=dynamic_frame,
            connection_type="s3",
            connection_options={"path": "s3://my-bucket/output/"},
            format="parquet"
        )

glueContext.forEachBatch(
    frame=kinesis_stream,
    batch_function=processBatch,
    options={
        "windowSize": "100 seconds",           # micro-batch window
        "checkpointLocation": "s3://my-bucket/checkpoints/job1/"
    }
)
```

### Key Configuration
- **windowSize:** How long Glue accumulates records before processing (e.g., `"100 seconds"`). Smaller = lower latency, more overhead.
- **checkpointLocation:** S3 path where Glue saves stream progress. Required for fault tolerance and restart recovery.

---

## 7. DynamicFrame Advanced Operations

### pushDownPredicate
A server-side filter applied **before** loading data into memory. Instead of reading all partitions and then filtering in Spark, Glue pushes the filter to the data catalog/S3 listing, loading only matching partitions.

```python
# Without pushDownPredicate: reads ALL partitions, then filters
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="my_db",
    table_name="sales"
)

# With pushDownPredicate: only loads partitions matching the predicate
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="my_db",
    table_name="sales",
    push_down_predicate="(year == '2024' and month == '12')"
)
```

**Benefit:** Dramatically reduces data read and cost. Only works on partitioned columns defined in the Glue catalog.

### resolveChoice: Handling Ambiguous Types
When Glue infers a column that has mixed types (e.g., some rows have integer `age`, others have string `"N/A"`), it creates a `choice` type. `resolveChoice` handles this ambiguity:

```python
# make_struct: keep both types as nested struct fields
dyf = dyf.resolveChoice(specs=[("age", "make_struct")])

# cast: force all values to a single type (incompatible values become null)
dyf = dyf.resolveChoice(specs=[("age", "cast:int")])

# project: keep only the specified type, drop others
dyf = dyf.resolveChoice(specs=[("age", "project:int")])
```

### toDF() and fromDF() Conversion Patterns
| Direction | Method | When to Use |
|-----------|--------|-------------|
| DynamicFrame → DataFrame | `dyf.toDF()` | When you need native Spark SQL operations, Window functions, complex joins not available in DynamicFrame API |
| DataFrame → DynamicFrame | `DynamicFrame.fromDF(df, glueContext, "name")` | When writing back via Glue sinks (S3, JDBC, Catalog), which require DynamicFrame |

**Pattern:** Use DynamicFrame for I/O (source/sink). Convert to DataFrame for complex transformations. Convert back to DynamicFrame to write.

---

## 8. Glue Connections

A Glue Connection stores credentials and network configuration needed to connect to a data source. Without a connection, Glue cannot reach resources inside a VPC or protected by credentials.

### JDBC Connections
Required fields when creating a JDBC connection:
- **JDBC URL:** e.g., `jdbc:mysql://mydb.cluster-xyz.us-east-1.rds.amazonaws.com:3306/myschema`
- **Username / Password** (stored securely, not in plain text in job script)
- **VPC:** The VPC where the data source lives
- **Subnet:** A subnet within that VPC (Glue ENI is placed here)
- **Security Group:** Applied to the Glue ENI — must allow outbound to the database port

### Why VPC Configuration is Required
Glue ETL workers are launched in AWS-managed infrastructure. To reach a database inside your VPC (RDS, Redshift), Glue creates an **Elastic Network Interface (ENI)** inside your subnet. This ENI is the actual network path to the data source.

- The database's **security group** must allow **inbound** from the Glue ENI's security group on the database port.
- If the database is in Account B and Glue is in Account A, **VPC Peering** is required so the ENI can reach the remote VPC.

### Connection Test
Before running a job, validate the connection from the Glue console:
1. Navigate to **AWS Glue → Data Connections**
2. Select your connection → **Test Connection**
3. Glue spins up a test runner in your VPC and attempts to connect
4. If it fails, check: security group rules, subnet routing, JDBC URL format, credentials

**Exam tip:** "Glue job fails to connect to RDS" → First check VPC/subnet/security group configuration in the Glue Connection, not the job script.

---

## 9. Glue Job Metrics in CloudWatch

All Glue jobs automatically emit metrics to CloudWatch under the `Glue` namespace.

### Key Metrics to Monitor

| Metric | What It Measures | Why It Matters |
|--------|-----------------|----------------|
| `glue.driver.aggregate.bytesRead` | Total bytes read from source | Understand I/O volume; high values may indicate missing pushdown predicates |
| `glue.driver.aggregate.recordsRead` | Total records read | Validate expected data volume; anomaly = pipeline issue upstream |
| `glue.ALL.jvm.heap.usage` | JVM heap memory utilization across all nodes | Values > 0.8 (80%) → risk of OOM errors; scale up worker type or increase workers |
| `glue.driver.aggregate.numFailedTasks` | Number of failed Spark tasks | Non-zero = data quality issues or resource pressure |
| `glue.driver.aggregate.elapsedTime` | Total job run time in milliseconds | Track job duration trends; sudden increases = data volume growth or skew |

### Setting CloudWatch Alarms
```
# Example: Alarm when Glue job runs longer than 30 minutes
Metric: glue.driver.aggregate.elapsedTime
Namespace: Glue
Statistic: Maximum
Threshold: 1800000 (ms = 30 minutes)
Action: SNS → ops team

# Example: Alarm when record count drops below expected minimum
Metric: glue.driver.aggregate.recordsRead
Statistic: Sum
Threshold: 10000 (expected minimum records)
Comparison: LessThanThreshold
Action: SNS → data engineering team
```

**Pattern:** Combine **job duration alarm** + **record count alarm** → catches both performance regressions and upstream data pipeline failures.

---

## When to Use vs Not Use

| Use Case | Verdict | Reason |
|----------|---------|--------|
| Serverless ETL | YES | No infra management, pay per second |
| Complex Transformations | YES | Spark power under the hood |
| Ultra Low Latency (<1s) | NO | Cold start 10-30s. Use Lambda or Flink |
| Long Running Custom Code | MAYBE | Expensive for 24/7 vs EMR |
| Schema discovery | YES | Crawlers + Data Catalog |
| Simple file conversion | NO | Lambda is cheaper for small event-driven tasks |

---

## Exam Gotchas

- Glue jobs have a **minimum billing of 1 minute** (1 DPU)
- Cold start can be 10-30 seconds - not suitable for sub-second latency
- **Glue vs EMR:** Glue is serverless + easy for pure ETL. EMR gives more control, better for constant heavy load and lower cost at massive scale with Spot instances
- Glue Crawlers are NOT free - only run them when needed, not on every file
- Bookmarks track previously processed data to avoid reprocessing
- **Glue Flex workers** run on spare capacity similar to Spot Instances — they can be reclaimed by AWS at any time. Do NOT use Flex for time-sensitive or SLA-bound jobs. Use G.1X or G.2X for reliability.
- **Job bookmarks track S3 objects by last modified time.** If you re-upload a file with the same name but new content, the bookmark will NOT detect it as new (same timestamp or slightly updated). You must **reset the bookmark** manually to reprocess it.
- **Python Shell jobs** have a maximum of 1 DPU (or 0.0625 DPU for small jobs). They are single-node and not distributed — NOT suitable for large datasets. Use Spark jobs for anything beyond a few GB.
- **Glue Data Quality rules are evaluated per row.** A rule like `IsComplete "email"` fires once for each row where email is null. If your ruleset has an action of FAIL, a single null in 1 million rows will fail the entire job unless you set a threshold (e.g., `IsComplete "email" >= 0.99`).

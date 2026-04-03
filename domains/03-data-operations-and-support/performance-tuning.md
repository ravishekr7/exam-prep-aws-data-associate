# Performance Tuning

Optimizing Spark, Glue, EMR, Redshift, Kinesis, and Lambda for data engineering workloads.

---

## Apache Spark (Glue / EMR)

### Data Skew

- One partition has 90% of data (e.g., `country=US`)
- One executor works forever, others finish instantly
- **Fixes:**
  - **Salting:** Add a random suffix to the skewed key, process, then aggregate
  - Choose a better partition key with higher cardinality
  - `repartition(n)` to force even redistribution (causes a full shuffle)
  - `skewHint` in Spark 3+ or Adaptive Query Execution (AQE)

### Small Files Problem

- Too many small files (KB-size) = excessive S3 LIST + OPEN overhead per file
- Metadata operations become the bottleneck, not compute
- **Fixes:**
  - `coalesce(n)` before writing — reduces partitions without full shuffle
  - `repartition(n)` — full shuffle, evenly distributes
  - Glue `groupFiles` option — combines small files on read
  - Kinesis Firehose buffering (time/size threshold before writing)
  - Post-hoc compaction job (periodic Spark job to merge small files)

### Optimal File Size

| Too small | Ideal | Too large |
|-----------|-------|-----------|
| < 32 MB | 128 MB – 1 GB | > 2 GB |
| Metadata overhead | Efficient S3 + Spark reads | Can't parallelize effectively |

### Shuffle Optimization

- Shuffles (joins, groupBy, distinct) are the most expensive Spark operations
- **Reduce shuffles:**
  - Use `broadcast join` for small tables (< 10 MB default, configurable up to ~2 GB)
  - Use `reduceByKey` instead of `groupByKey` (partial aggregation before shuffle)
  - Avoid `distinct()` on large datasets — use approximate counting (`approx_count_distinct`)
- **Tune shuffle partitions:**
  ```python
  spark.conf.set("spark.sql.shuffle.partitions", "400")  # default is 200
  # Rule of thumb: target 128 MB per partition after shuffle
  ```

### Adaptive Query Execution (AQE) — Spark 3+

- AQE dynamically re-plans queries at runtime based on actual data statistics
- **Auto-coalesces** small shuffle partitions → reduces task count after a join
- **Converts sort-merge join to broadcast join** when a relation turns out to be small
- **Skew join optimization** — splits skewed partitions automatically
- Enable in Glue 4.0+ and EMR 6.x+:
  ```python
  spark.conf.set("spark.sql.adaptive.enabled", "true")
  spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
  ```

### Cache and Persist

```python
# Cache in memory (recomputed if evicted)
df.cache()

# Persist with storage level
from pyspark import StorageLevel
df.persist(StorageLevel.MEMORY_AND_DISK)

# Release when done
df.unpersist()
```

- Use when a DataFrame is used multiple times in the same job
- `MEMORY_AND_DISK`: Spills to disk if memory is insufficient — good for large DataFrames

---

## AWS Glue Specific Tuning

### Worker Types

| Worker Type | vCPUs | Memory | Use Case |
|-------------|-------|--------|---------|
| **Standard** | 2 | 16 GB | Legacy; basic jobs |
| **G.1X** | 4 | 16 GB | Memory-efficient workloads, default |
| **G.2X** | 8 | 32 GB | Memory-intensive jobs (large joins, wide DataFrames) |
| **G.4X** | 16 | 64 GB | Large-scale ML, very wide transforms |
| **G.8X** | 32 | 128 GB | Largest workloads |
| **Z.2X** | 8 | 64 GB | Ray workloads |

> Use G.2X when jobs fail with out-of-memory errors or have many wide DataFrame operations.

### DPU Autoscaling

- Enable **auto-scaling** on Glue 3.0+ jobs — Glue adjusts DPUs dynamically during execution
- No need to over-provision; scales up during heavy shuffles, down during idle phases
- Set `--enable-auto-scaling` job parameter
- Review CloudWatch metric `glue.ALL.jvm.heap.usage` to detect memory pressure

### Job Bookmarks

- Track which data has already been processed (incremental loads)
- Glue remembers the last processed S3 key or DynamoDB timestamp
- Enable per job: `--job-bookmark-option job-bookmark-enable`
- Use `commit()` to advance the bookmark only after successful processing

```python
job.init(args['JOB_NAME'], args)
# ... processing logic ...
job.commit()  # advances the bookmark
```

### Glue Pushdown Predicates

- Filter data at the catalog level before loading into Spark
- Avoids reading unnecessary partitions:
  ```python
  datasource = glueContext.create_dynamic_frame.from_catalog(
      database="mydb",
      table_name="events",
      push_down_predicate="year='2024' and month='01'"
  )
  ```

### Glue Connection Reuse

- Set `--enable-continuous-cloudwatch-log` for real-time logs during job execution
- Use `--TempDir` S3 path for shuffle spill — ensure adequate S3 permissions
- Connection pooling for JDBC sources: configure `--connection-num-partitions`

---

## Amazon EMR Specific Tuning

### Instance Fleet vs Instance Groups

| Feature | Instance Groups | Instance Fleets |
|---------|----------------|-----------------|
| **Composition** | One instance type per group | Mix of instance types per fleet |
| **Spot handling** | Manual replacement | Automatic diversification across types |
| **Scaling** | Manual or managed | Managed scaling with diversification |
| **Use case** | Predictable workloads | Cost-optimized Spot usage |

> **Instance Fleets** are preferred for cost optimization — EMR diversifies across instance types to maximize Spot availability.

### EMR Managed Scaling

- EMR automatically adds/removes core and task nodes based on YARN metrics
- Responds to `YARNMemoryAvailablePercentage` and `ContainerPendingRatio`
- Set minimum and maximum node counts; EMR stays within bounds
- Available for both Instance Groups and Instance Fleets

### EMR on EC2 vs EMR Serverless

| Attribute | EMR on EC2 | EMR Serverless |
|-----------|-----------|----------------|
| **Provisioning** | Manual cluster creation | Fully managed, no cluster |
| **Idle cost** | Yes (cluster stays up) | None (pay per job) |
| **Startup** | Cluster startup time | Pre-initialized workers reduce latency |
| **Use case** | Long-running clusters, interactive | Batch jobs, variable workloads |

### Spark Configuration on EMR

```bash
# Key parameters to tune
--conf spark.executor.memory=10g
--conf spark.executor.cores=4
--conf spark.driver.memory=5g
--conf spark.sql.shuffle.partitions=400
--conf spark.sql.adaptive.enabled=true
--conf spark.dynamicAllocation.enabled=true
--conf spark.dynamicAllocation.minExecutors=2
--conf spark.dynamicAllocation.maxExecutors=50
```

### Bootstrap Actions

- Scripts that run on all nodes before Hadoop starts
- Use to install additional libraries, configure system settings
- Stored in S3:
  ```bash
  aws emr create-cluster \
    --bootstrap-actions Path=s3://my-bucket/install-libs.sh
  ```

### EMRFS Consistent View

- S3 is eventually consistent for LIST operations — EMRFS adds metadata tracking in DynamoDB
- Ensures Spark reads all files written in a prior step
- Enable with `--emrfs Consistent=true`
- Less necessary since S3 Strong Consistency (2021) but still relevant for multi-step jobs

---

## Redshift Performance

### Distribution Styles

| Style | How Data Distributes | Use When |
|-------|---------------------|---------|
| **KEY** | By values in a column | Frequently joined columns → collocate matching rows |
| **ALL** | Full copy on every node | Small dimension tables |
| **EVEN** | Round-robin | No clear join key; used for fact tables when KEY isn't ideal |
| **AUTO** | Redshift chooses | Default; good starting point |

### Sort Keys

- **Compound sort key:** Most beneficial for queries filtering on leading columns in order. Use this for all new tables.
- **Interleaved sort key:** ~~Equal weight to all columns~~ — **Deprecated by AWS.** Do not create new tables with interleaved sort keys. `VACUUM REINDEX` (required to maintain their effectiveness) is prohibitively slow on large tables. For multi-column filter patterns, use compound sort key on the most commonly filtered column.
- Zone map pruning: Redshift skips 1 MB blocks where values are outside filter range

### Query Plan Analysis (EXPLAIN)

```sql
EXPLAIN SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id
WHERE order_date > '2024-01-01';
```

Key things to look for in the plan:
- `DS_BCAST_INNER` — Broadcasting inner table (small table: OK; large table: fix the distribution key)
- `DS_DIST_BOTH` — Both tables redistributed (expensive — fix distribution key mismatch)
- `DS_DIST_NONE` — No redistribution needed (best case)
- `MERGE JOIN` vs `HASH JOIN` — MERGE is faster (requires sorted inputs)

### Workload Management (WLM)

- Routes queries to **queues** with allocated memory and concurrency slots
- Prevents large ETL queries from blocking dashboards

```
Queue 1 (ETL): 60% memory, 3 concurrency slots, label="etl"
Queue 2 (BI):  30% memory, 10 concurrency slots, label="bi"
Default queue: 10% memory, remaining slots
```

- **Automatic WLM:** Redshift dynamically adjusts memory and concurrency (recommended)
- **Manual WLM:** You define memory and concurrency per queue
- Set query group via `SET query_group TO 'etl';` to route to the correct queue

### Compression

```sql
-- Let Redshift recommend compression encodings
ANALYZE COMPRESSION my_table;

-- Apply recommended encodings (requires table recreation)
CREATE TABLE my_table_compressed (
  id INT ENCODE az64,
  event_time TIMESTAMP ENCODE az64,
  status VARCHAR(20) ENCODE zstd
);
```

- `AZ64` — Best for integer/date columns
- `ZSTD` — Good general-purpose compression
- `LZO` — Fast decompression for string columns
- `RAW` — No compression (use for sort key columns)

### VACUUM and ANALYZE

```sql
-- Reclaim space + re-sort unsorted rows
VACUUM my_table;

-- Full sort + reclaim (expensive — run during off-hours)
VACUUM FULL my_table;

-- Update query planner statistics
ANALYZE my_table;
ANALYZE my_table(column1, column2);
```

- Automatic VACUUM runs in background but may not keep up with heavy writes
- Run manual VACUUM after bulk loads or large deletes
- `ANALYZE` is fast — run after significant data changes

### Redshift Spectrum (External Tables)

```sql
-- Query S3 data directly without loading
SELECT * FROM spectrum_schema.external_table
WHERE year='2024' AND month='01';
```

- Pushes predicates and aggregations to Spectrum layer (runs on thousands of nodes)
- Use Parquet + partition projection for best performance
- Combine with Redshift tables using JOINs (join large S3 fact tables with Redshift dimensions)

### Concurrency Scaling

- Adds transient clusters when main cluster queue depth exceeds threshold
- Free credits: 1 hour per 24 hours of main cluster usage
- Configure which queues use concurrency scaling in WLM settings

---

## Kinesis Performance

### Shard Capacity Limits

| Metric | Per Shard Limit |
|--------|----------------|
| Write (PutRecord) | 1 MB/s or 1,000 records/s |
| Read (GetRecords) | 2 MB/s |
| Read with Enhanced Fan-Out | 2 MB/s **per consumer** |

### IteratorAge (Consumer Lag)

- High `IteratorAge` = consumer is processing slower than data arrives
- Target: keep IteratorAge close to 0
- **Fixes:**
  - Increase shard count (re-shard)
  - Increase Lambda batch size
  - Optimize Lambda processing logic
  - Use Enhanced Fan-Out (dedicated 2 MB/s per consumer)

### Lambda Batch Window Optimization

When Lambda polls Kinesis:
- **Batch size:** Max records per invocation (up to 10,000)
- **Batch window:** Wait up to N seconds to fill a batch before invoking
- Larger batch size = fewer Lambda invocations = lower cost but higher latency
- Use `BisectBatchOnFunctionError=true` to automatically split a failing batch to isolate bad records

```
Low latency:  BatchSize=100, BisectOnError=true
High throughput: BatchSize=10000, BatchWindow=60s
```

### Enhanced Fan-Out

- Dedicated 2 MB/s per registered consumer (not shared 2 MB/s)
- Uses HTTP/2 push model — lower latency than `GetRecords` polling
- Use when multiple Lambda functions or applications consume the same stream
- Register consumer: `aws kinesis register-stream-consumer`

### On-Demand vs Provisioned Mode

| Attribute | On-Demand | Provisioned |
|-----------|----------|-------------|
| Scaling | Automatic | Manual shard management |
| Cost | Higher per GB | Lower per GB at steady load |
| Use case | Unpredictable traffic | Predictable, high volume |

---

## Lambda Performance

### Memory and CPU

- Lambda allocates CPU proportional to memory (1,769 MB = 1 full vCPU)
- More memory = faster CPU = shorter execution time = may reduce cost despite higher memory price
- Test at 512 MB, 1024 MB, 3008 MB — plot cost (memory × duration) to find the sweet spot
- Use **AWS Lambda Power Tuning** (open source Step Functions workflow) to automate this

### Cold Starts

- Cold start = Lambda initializes a new execution environment
- Causes: first invocation, scaling out, after idle period
- **Reduce cold starts:**
  - **Provisioned Concurrency:** Pre-initialize N execution environments — eliminates cold starts
  - Keep deployment package small (use Lambda Layers for large dependencies)
  - Avoid heavy imports at module level (lazy load inside handlers)
  - Use ARM64 (Graviton2) — faster startup + lower cost

### Connection Pooling

Avoid creating DB connections inside the handler — move them to module level:

```python
# Bad: new connection per invocation
def handler(event, context):
    conn = psycopg2.connect(...)

# Good: reuse connection across warm invocations
import psycopg2
conn = psycopg2.connect(...)  # module-level

def handler(event, context):
    # reuse conn
```

### Ephemeral Storage (/tmp)

- Default: 512 MB; configurable up to 10 GB
- Use for temporary file processing (unzip, transform, re-upload to S3)
- Persists across warm invocations (within the same execution environment)
- Cost: $0.0000000309 per GB-second above 512 MB

### Lambda Layers

- Package shared libraries once, attach to multiple functions
- Up to 5 layers per function; total unzipped size (function + layers) < 250 MB
- Use for: pandas, PyArrow, psycopg2, boto3 upgrades

---

## Performance Debugging Guide

### Slow Glue Job

1. Check `glue.ALL.jvm.heap.usage` — if > 80%, upgrade to G.2X workers
2. Check CloudWatch logs for GC (garbage collection) overhead
3. Enable job bookmarks — ensure you're not reprocessing data
4. Check for data skew — look for task duration variance in Spark UI
5. Verify AQE is enabled (Glue 3.0+)

### Slow Athena Query

1. Check bytes scanned — partitioning and file format reduce scan size
2. Use `EXPLAIN` to see query plan
3. Check for full table scans (missing partition filter)
4. Ensure Parquet/ORC columnar format (not CSV)
5. Check for too many small files — compaction may be needed

### Slow Redshift Query

1. Run `EXPLAIN` — look for `DS_BCAST_INNER` or `DS_DIST_BOTH`
2. Check `STL_QUERY` table for actual execution details
3. Check `SVV_TABLE_INFO` for unsorted rows percentage
4. Verify WLM queue assignment — query may be in wrong queue
5. Run `ANALYZE` if statistics are stale

### Kinesis High IteratorAge

1. Check Lambda `Duration` and `Throttles` metrics
2. Increase `BatchSize` to process more records per invocation
3. Check for poison pill records causing repeated failures
4. Consider increasing shard count if throughput is the bottleneck

---

## Backup and Disaster Recovery

### RPO vs RTO

- **RPO (Recovery Point Objective):** How much data can you afford to lose? (measured in time)
- **RTO (Recovery Time Objective):** How fast must you recover operations?

### Service DR Strategies

| Service | RPO | RTO | Strategy |
|---------|-----|-----|---------|
| **S3** | 0 (versioning) | Minutes | Versioning + Cross-Region Replication (CRR) |
| **DynamoDB** | Seconds (PITR) | Minutes | PITR (35-day window, per-second granularity) + Global Tables |
| **RDS** | Minutes | Minutes | Automated snapshots (1-35 days) + Multi-AZ standby |
| **Redshift** | Hours | Hours | Automated snapshots + Cross-region snapshot copy |
| **Aurora** | < 1 second | < 1 min | Aurora Global Database (< 1s replication) |

### Cross-Region Replication Patterns

```
Primary Region: S3 bucket → CRR → DR Region: S3 bucket
                                  ↓
                             Glue Catalog (manual sync or Catalog replication)
```

- S3 CRR replicates objects only — Glue Catalog metadata must be replicated separately
- DynamoDB Global Tables: active-active, any region can handle reads and writes

---

## Exam Gotchas

- **AQE** (Adaptive Query Execution) — enabled in Glue 3.0+ and EMR 6.x — auto-optimizes skew and joins
- **`spark.sql.shuffle.partitions`** default is 200 — tune based on data size (target ~128 MB/partition)
- **G.2X workers** — answer for Glue out-of-memory errors
- **Job bookmarks** — Glue incremental processing without tracking state yourself
- **Instance Fleets** — preferred for EMR Spot usage (diversifies across types)
- **EMR Managed Scaling** — auto-adjusts cluster size based on YARN metrics
- **`DS_BCAST_INNER` in Redshift EXPLAIN** — inner table being broadcast = distribution key mismatch
- **WLM queues** — prevent ETL from starving BI queries in Redshift
- **Provisioned Concurrency** — only way to fully eliminate Lambda cold starts
- **Lambda memory and CPU are coupled** — more memory = more CPU = potentially faster and cheaper
- **Enhanced Fan-Out** — dedicated 2 MB/s per consumer; use when multiple apps consume same Kinesis stream
- **`TreatMissingData=breaching`** — set on CloudWatch Alarms when a metric going silent means a problem
- **Connection pooling in Lambda** — always initialize DB connections at module level, not inside handler
- RPO/RTO: Aurora Global Database has the lowest RPO (< 1 second cross-region replication)

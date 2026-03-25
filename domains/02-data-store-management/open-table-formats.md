# Open Table Formats and Vector Concepts

Covers transactional data lake formats and vector database concepts.

---

## Open Table Formats

### The Problem
Standard S3 files (Parquet, CSV) are **immutable**. You cannot UPDATE or DELETE individual rows without rewriting the entire file. This is a problem for:
- GDPR "Right to be Forgotten" (delete specific user data)
- Late-arriving data corrections
- Slowly Changing Dimensions (SCD)

---

## Apache Iceberg

### Core Concept
Iceberg is an **open table format specification** — not a storage engine. It defines how table metadata, data files, and transaction logs are organized on any object storage (S3, HDFS, GCS). Iceberg itself doesn't store or process data; it provides the protocol that query engines (Spark, Flink, Athena, Trino) use to read and write tables atomically.

**AWS's preferred open table format** with deep native integration: Athena, EMR, Glue, Redshift Spectrum, and S3 Tables all support Iceberg natively.

### ACID Transactions
Iceberg uses **snapshot isolation**:
- Every write (INSERT, UPDATE, DELETE, MERGE) creates a new **snapshot** — an immutable point-in-time view of the table
- Readers always see a consistent snapshot; they are never blocked by in-progress writes
- Writers don't block readers — multiple operations can proceed concurrently

### Time Travel
Query historical versions of the table:
```sql
-- Query as of a specific snapshot ID:
SELECT * FROM my_catalog.my_db.orders
FOR VERSION AS OF 5432198765432198765;

-- Query as of a specific timestamp:
SELECT * FROM my_catalog.my_db.orders
FOR TIMESTAMP AS OF TIMESTAMP '2024-01-15 00:00:00';

-- In Athena:
SELECT * FROM "my_db"."orders"
FOR TIMESTAMP AS OF TIMESTAMP '2024-01-15 00:00:00 UTC';
```

### Partition Evolution
Change the partitioning strategy of a table **without rewriting any existing data**. New partitions use the new strategy; old data files keep their original partitions.

```sql
-- Table originally partitioned by year and month
ALTER TABLE orders ADD PARTITION FIELD days(order_date);
-- Going forward, data is partitioned by day. Old data still partitioned by year/month.
-- Queries work correctly across both partition schemes transparently.
```

### Schema Evolution
Add, drop, rename, or reorder columns **without rewriting data files**. The schema is tracked in metadata; queries adapt automatically.
- **Add column:** New column reads as NULL for existing rows
- **Drop column:** Dropped column is excluded from future reads (data remains in files but is not returned)
- **Rename column:** All readers see the new name; data files unchanged

### Hidden Partitioning
Iceberg handles partition transforms automatically. You never need to include a partition column explicitly in your queries — Iceberg rewrites the query to use partition pruning automatically.

```sql
-- In Hive-style partitioning: you must filter on the partition column
WHERE year = '2024' AND month = '01'

-- With Iceberg hidden partitioning: just filter on the actual column
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
-- Iceberg automatically prunes partitions using its metadata
```

### Compaction
The small file problem: frequent small writes (streaming, per-record inserts) create millions of tiny Parquet files. This degrades query performance because every file requires an S3 API call to open.

Iceberg solves this with periodic compaction that merges small files into larger ones:

```python
# In AWS Glue: use the rewriteDataFiles action
from awsglue.context import GlueContext
from awsglue.dynamicframe import DynamicFrame

# Compact small files in an Iceberg table
spark.sql("""
    CALL my_catalog.system.rewrite_data_files(
        table => 'my_db.orders',
        options => map(
            'target-file-size-bytes', '134217728',  -- 128 MB target
            'min-input-files', '5'                  -- compact when >= 5 small files
        )
    )
""")
```

### AWS Native: Amazon S3 Tables
AWS announced **S3 Tables** (late 2024): a fully managed Iceberg table service built on S3. AWS handles all compaction, snapshot management, and metadata maintenance automatically — no manual operations required.

---

## Apache Hudi

### Primary Use Case
Hudi excels at **incremental data processing** and **Change Data Capture (CDC)** pipelines. If your primary workload is upserts (insert + update in one operation) from a CDC source (DMS, Debezium), Hudi's native upsert capability makes it a strong choice.

### Copy-on-Write (COW)
Every write **rewrites** the affected Parquet files in full.
- **Read latency:** Low — data files are always clean, optimized Parquet files. Readers see standard Parquet files with no merge required.
- **Write latency:** Higher — even a single-row update rewrites an entire Parquet file (which may be 128 MB or more).
- **Best for:** Read-heavy analytics workloads where query performance matters more than write speed.

### Merge-on-Read (MOR)
Writes append **delta log files** (small Avro or Parquet files) alongside the base Parquet files. Base files are only rewritten during compaction.
- **Read latency:** Higher for "snapshot" reads — readers must merge base files + delta logs at query time.
- **Write latency:** Very low — writes are just appends to a log file.
- **Best for:** Write-heavy CDC workloads where writes happen frequently and compaction can run on a schedule.

### Decision Guide
```
Is the workload read-heavy (analysts querying dashboards)?
  YES → Copy-on-Write (COW): clean files, fast reads

Is the workload write-heavy (CDC from Oracle, 10k updates/min)?
  YES → Merge-on-Read (MOR): fast writes, compact periodically during off-hours
```

### Hudi Metadata Table
Optional but recommended for large datasets. The Hudi metadata table stores file listings and other metadata in a Hudi table itself, eliminating the need for S3 LIST operations on large datasets. S3 LIST is slow and expensive when a table has millions of files.

---

## Delta Lake

### Transaction Log
Delta Lake's source of truth is the **`_delta_log/` directory** in the table's S3 location. It contains an ordered sequence of JSON files (and periodic checkpoint Parquet files) that record every operation performed on the table.

```
s3://my-bucket/my-table/
├── _delta_log/
│   ├── 00000000000000000000.json    # Initial CREATE TABLE
│   ├── 00000000000000000001.json    # First INSERT
│   ├── 00000000000000000002.json    # UPDATE 500 rows
│   ├── 00000000000000000010.json    # Checkpoint (compact first 10 logs)
│   └── ...
├── part-00000-abc123.parquet
├── part-00001-def456.parquet
└── ...
```

Reading a Delta table = reading the transaction log to find the current set of valid data files, then reading those files.

### OPTIMIZE
Compacts small files into larger ones using bin-packing. Run periodically to maintain query performance.
```sql
OPTIMIZE my_catalog.my_db.orders;
-- After OPTIMIZE, small files are merged into ~1 GB files
```

### VACUUM
Removes data files that are no longer referenced by the transaction log (old versions, deleted files). Default retention: 7 days.
```sql
VACUUM my_catalog.my_db.orders RETAIN 168 HOURS;  -- 7 days
-- Files older than retention period are permanently deleted from S3
```
> **Warning:** Do not set retention below 7 days if other jobs may be reading historical snapshots.

### Z-ORDER
Multi-dimensional data clustering applied within OPTIMIZE. Co-locates rows with similar values across multiple columns within each data file. Dramatically improves query performance when filtering on multiple columns.
```sql
-- Co-locate data by customer_id and product_category within each file
OPTIMIZE my_catalog.my_db.orders
ZORDER BY (customer_id, product_category);
-- Queries with WHERE customer_id = X AND product_category = Y now skip most files
```

### Delta Live Tables
Declarative pipeline framework on top of Delta Lake (primarily Databricks). You define datasets and their dependencies; the framework handles incremental processing, error handling, and data quality expectations automatically. Most relevant for Databricks users; Delta Lake itself (without DLT) can run on EMR.

---

## Open Table Format Comparison

| Feature | Apache Iceberg | Apache Hudi | Delta Lake |
|---------|---------------|-------------|------------|
| **ACID transactions** | Yes (snapshot isolation) | Yes | Yes |
| **Time travel** | Yes (snapshot ID or timestamp) | Yes (commit timeline) | Yes (version number or timestamp) |
| **Schema evolution** | Strong (add/drop/rename/reorder) | Moderate (add columns) | Good (add/rename columns) |
| **Upserts / deletes** | Yes (MERGE INTO, DELETE, UPDATE) | Yes (primary key upserts, native) | Yes (MERGE INTO, DELETE, UPDATE) |
| **Compaction** | rewriteDataFiles action; S3 Tables does it automatically | Scheduled compaction (COW) or automatic (MOR) | OPTIMIZE command |
| **AWS native support** | Best — Athena, Glue, EMR, Redshift Spectrum, S3 Tables | Good — EMR, Glue (with connector) | EMR Spark only (no native Athena support) |
| **Primary use case** | General data lake, S3-native analytics, GDPR compliance | CDC pipelines, upserts from operational databases | Spark-based data lakes (especially Databricks) |
| **Exam relevance** | **Highest** — AWS's recommended format | Medium | Lower (mostly EMR/Databricks scenarios) |

---

## Amazon S3 Tables

Announced in late 2024, S3 Tables is a **fully managed, serverless Iceberg table service** built on Amazon S3. It extends the S3 API with native Iceberg table semantics.

### What It Does
- **Automatic compaction:** AWS automatically merges small files in the background. No manual `rewriteDataFiles` jobs needed.
- **Automatic snapshot management:** Old snapshots are automatically expired and cleaned up.
- **Automatic metadata maintenance:** Table statistics and metadata are kept current.

### Native Integration
- **Amazon Athena:** Query S3 Tables directly using SQL
- **Amazon EMR:** Use Spark with S3 Tables as a catalog
- **AWS Glue:** ETL jobs can read and write S3 Tables

### When to Choose S3 Tables
- You want Iceberg capabilities (ACID, time travel, schema evolution) with **zero operational overhead**
- You don't want to manage compaction schedules, VACUUM jobs, or snapshot expiry
- New projects starting in 2025+ where S3 Tables is the obvious choice for managed Iceberg

**Exam tip:** "Which option provides the **least operational overhead** for managing an Iceberg data lake on AWS?" → **Amazon S3 Tables**.

---

## Vector Concepts (New for Exam)

### Vector Index Types

**HNSW (Hierarchical Navigable Small Worlds)**
- Graph-based approximate nearest neighbor search
- Fast query performance
- Higher memory usage
- Supported in: Aurora PostgreSQL (pgvector)

**IVF (Inverted File Index)**
- Cluster-based approximate nearest neighbor search
- Lower memory usage than HNSW
- Slightly lower recall
- Good for large-scale vector datasets

### Vectorization in AWS
- **Amazon Bedrock Knowledge Base:** Stores and retrieves vector embeddings
- **Aurora PostgreSQL with pgvector:** Vector similarity search alongside relational data
- **Amazon OpenSearch Service:** Vector search capabilities

---

## Exam Gotchas

- **"Row-level delete/update on S3"** = Open table format (Iceberg is the default answer)
- **"GDPR right to be forgotten in data lake"** = Iceberg/Hudi
- **"Time travel"** or **"query historical data versions"** = Iceberg
- Apache Iceberg has the deepest AWS integration - prefer it in exam answers
- **"Vector similarity search"** = HNSW index type
- Know that vectorization concepts exist (Bedrock, Aurora pgvector) but deep ML knowledge is out of scope
- **"Least operational overhead for Iceberg on AWS"** = Amazon S3 Tables (automatic compaction and maintenance)
- **Hudi MOR** is the right choice when writes are very frequent (CDC) and you can tolerate slightly slower reads until compaction runs
- **Delta Lake** has limited native AWS service support — prefer Iceberg for any scenario involving Athena or Redshift Spectrum

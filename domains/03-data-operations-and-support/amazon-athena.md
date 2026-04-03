# Amazon Athena

Serverless interactive query service to analyze data in S3 using standard SQL.

---

## Key Features

- **Serverless:** No infrastructure to manage
- **Pay per query:** $5 per TB of data scanned (DDL and failed queries = no charge)
- **SQL Engine:** Based on Trino (v3) / Presto (v2)
- **Data Sources:** S3 (primary) + 25+ federated connectors
- **Formats:** CSV, JSON, Parquet, ORC, Avro, Ion
- **Catalog:** Uses Glue Data Catalog for schema metadata

---

## Athena Engine Versions

| Feature | Engine v2 (Presto) | Engine v3 (Trino) |
|---------|--------------------|--------------------|
| **SQL standard** | ANSI SQL (partial) | Full ANSI SQL + extensions |
| **Performance** | Baseline | Generally faster (better optimizer) |
| **Window functions** | Basic | Extended (LEAD, LAG, NTILE, etc.) |
| **MERGE statement** | No | Yes (v3.0+) |
| **MATCH_RECOGNIZE** | No | Yes (pattern matching in sequences) |
| **JSON functions** | Limited | Richer JSON support |
| **Recommended** | Legacy workloads | Default for new workgroups |

> **Exam tip:** Engine v3 is the default for new workgroups. Use v2 only for compatibility with existing queries.

---

## Workgroups

Workgroups provide **resource isolation, cost control, and access management** per team or use case.

### What Workgroups Control

| Setting | Purpose |
|---------|---------|
| **Query result location** | Force results to specific S3 path |
| **Encryption** | SSE-S3 or SSE-KMS for results |
| **Bytes scanned limit (per query)** | Cancel queries that would scan too much data |
| **Bytes scanned limit (per workgroup/day)** | Cap daily spend per team |
| **Engine version** | v2 or v3 per workgroup |
| **Query result reuse** | Enable/disable result caching |

### Example Workgroup Scenarios

```
Workgroup: "data-science-team"
  - Engine: v3
  - Result location: s3://results-bucket/data-science/
  - Per-query bytes limit: 10 GB (cancel expensive ad-hoc queries)
  - Encryption: SSE-KMS

Workgroup: "production-etl"
  - Engine: v3
  - Result location: s3://results-bucket/etl/
  - Per-workgroup daily limit: 1 TB
```

- Assign IAM permissions per workgroup — teams can only use their workgroup
- Default workgroup is used if none is specified
- Workgroup metrics (bytes scanned, query count) flow to CloudWatch

> **Exam tip:** Workgroups = cost control + access isolation. If the question asks how to prevent one team's queries from scanning too much data, the answer is workgroup per-query byte limit.

---

## Partitioning

### Standard Partitioning (Hive-style)

```sql
-- Partition columns are defined in the table schema
CREATE EXTERNAL TABLE events (
  event_id STRING,
  user_id  STRING,
  payload  STRING
)
PARTITIONED BY (year STRING, month STRING, day STRING)
STORED AS PARQUET
LOCATION 's3://my-bucket/events/';

-- Load partitions from S3 structure
MSCK REPAIR TABLE events;
-- OR add manually:
ALTER TABLE events ADD PARTITION (year='2024', month='01', day='15')
  LOCATION 's3://my-bucket/events/year=2024/month=01/day=15/';
```

- `MSCK REPAIR TABLE` scans S3 for partition folders and registers them in the Glue catalog
- Slow on large tables with many partitions — use partition projection instead

### Partition Projection

Athena computes partitions at query time — **no Glue Crawler or MSCK REPAIR needed**.

```sql
CREATE EXTERNAL TABLE events (
  event_id STRING,
  user_id  STRING
)
PARTITIONED BY (year STRING, month STRING, day STRING)
STORED AS PARQUET
LOCATION 's3://my-bucket/events/'
TBLPROPERTIES (
  "projection.enabled"            = "true",
  "projection.year.type"          = "integer",
  "projection.year.range"         = "2022,2030",
  "projection.month.type"         = "integer",
  "projection.month.range"        = "1,12",
  "projection.month.digits"       = "2",
  "projection.day.type"           = "integer",
  "projection.day.range"          = "1,31",
  "projection.day.digits"         = "2",
  "storage.location.template"     = "s3://my-bucket/events/year=${year}/month=${month}/day=${day}"
);
```

### Partition Projection Types

| Type | Example Range | Use Case |
|------|--------------|---------|
| `integer` | `"1,31"` | Day, month, numeric IDs |
| `date` | `"2022-01-01,NOW"` | Date columns (auto-advances with NOW) |
| `enum` | `"us-east-1,eu-west-1,ap-south-1"` | Regions, environments, status codes |
| `injected` | N/A | Value passed directly in the query WHERE clause |

> **Exam tip:** Partition projection eliminates the need for Glue Crawlers when your partition scheme is predictable (dates, integers, enums).

---

## CTAS (Create Table As Select)

```sql
-- Convert CSV to Parquet with partitioning
CREATE TABLE events_parquet
WITH (
  format           = 'PARQUET',
  external_location = 's3://my-bucket/events-parquet/',
  partitioned_by   = ARRAY['year', 'month'],
  bucketed_by      = ARRAY['user_id'],
  bucket_count     = 10
)
AS SELECT *, year(event_time) AS year, month(event_time) AS month
FROM events_csv;
```

- Results written to S3 as a new external table
- Useful for: format conversion, creating aggregated tables, pre-materializing expensive queries
- **Unload** is an alternative for writing query results to S3 without creating a table:
  ```sql
  UNLOAD (SELECT * FROM events WHERE year = '2024')
  TO 's3://my-bucket/output/'
  WITH (format = 'PARQUET', compression = 'SNAPPY');
  ```

---

## Prepared Statements

Parameterized queries that prevent SQL injection and improve reusability:

```sql
-- Prepare once
PREPARE get_events FROM
  SELECT * FROM events WHERE region = ? AND year = ?;

-- Execute with parameters
EXECUTE get_events USING 'us-east-1', '2024';

-- Deallocate when done
DEALLOCATE PREPARE get_events;
```

- Parameters are bound at execution time — safe from SQL injection
- Prepared statements are session-scoped (not persisted between API calls)
- Useful in application code where the same query structure runs with different values

---

## Query Result Reuse (Caching)

- Reuse results from a previous identical query (same SQL + same data)
- Configurable window: up to 7 days
- Enable at workgroup level or per query:
  ```sql
  -- Via API: set ResultReuseConfiguration
  MaxAgeInMinutes: 60
  ```
- Cached results incur **no scan cost** — pay $0 for a cache hit
- Cache invalidated when underlying data changes (Athena checks table modification time)

> **Exam tip:** Result reuse = zero scan cost for repeated identical queries. Ideal for dashboards running the same query every few minutes.

---

## Federated Query

Query data across multiple sources without moving it.

### How It Works

```
Athena SQL → Lambda connector → Source (RDS, DynamoDB, Redshift, CloudWatch, etc.)
                ↓
         Results returned and joined with S3 data
```

- Each connector is a **Lambda function** deployed in your account
- Results from external sources can be **joined with S3 tables**
- Pre-built connectors: DynamoDB, RDS (MySQL/PostgreSQL/SQL Server), Redshift, CloudWatch Logs, DocumentDB, HBase, Redis, CMDB, JDBC-generic

### Federated Query Limits

- External data is pulled through Lambda — limited by Lambda memory and network
- Not suitable for large-scale bulk reads from external sources
- Use for: selective lookups, enrichment joins, ad-hoc cross-source analysis

### Spill Location

- Lambda connectors write intermediate data to S3 ("spill bucket") for large result sets
- Required configuration: specify a spill S3 bucket when deploying the connector

---

## Athena Notebooks (Apache Spark)

- Run Spark code interactively against S3 data — no cluster to manage
- Jupyter notebook interface in the Athena console
- Sessions are ephemeral — Spark environment created on demand, terminated when idle

### Session Configuration

```python
# Configure Spark session settings
spark.conf.set("spark.sql.shuffle.partitions", "200")
spark.conf.set("spark.executor.memory", "4g")
```

- Session types: Spark-based (interactive notebooks) vs SQL-based (standard Athena)
- Notebooks stored in S3; share by granting S3 access
- IAM execution role controls what data the Spark session can access

---

## Athena with Apache Iceberg

Athena has native support for Apache Iceberg tables — enabling ACID DML (INSERT, UPDATE, DELETE, MERGE), time travel, and schema evolution directly in SQL without Spark or EMR.

### Create an Iceberg Table in Athena

```sql
CREATE TABLE my_catalog.orders (
    order_id   BIGINT,
    customer_id INT,
    amount     DECIMAL(10,2),
    status     VARCHAR(20),
    order_date DATE
)
LOCATION 's3://my-bucket/iceberg/orders/'
TBLPROPERTIES ('table_type'='ICEBERG', 'format'='parquet');
```

### DML Operations

```sql
-- INSERT new rows
INSERT INTO orders VALUES (1001, 42, 99.99, 'PENDING', DATE '2024-01-15');

-- UPDATE specific rows (not possible on plain Parquet!)
UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1001;

-- DELETE specific rows (GDPR right-to-erasure)
DELETE FROM orders WHERE customer_id = 42;

-- MERGE INTO (upsert — combine INSERT + UPDATE in one statement)
MERGE INTO orders t
USING staging_orders s ON t.order_id = s.order_id
WHEN MATCHED THEN
    UPDATE SET status = s.status, amount = s.amount
WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, amount, status, order_date)
    VALUES (s.order_id, s.customer_id, s.amount, s.status, s.order_date);
```

### Time Travel Queries

```sql
-- Query table as of a specific timestamp
SELECT * FROM orders
FOR TIMESTAMP AS OF TIMESTAMP '2024-01-01 00:00:00 UTC';

-- Query table as of a specific snapshot ID
SELECT * FROM orders
FOR VERSION AS OF 5432198765432198765;
```

### Maintenance Operations (Athena SQL)

```sql
-- OPTIMIZE: compact small files into larger ones (solves small files problem)
OPTIMIZE orders REWRITE DATA USING BIN_PACK;

-- VACUUM: remove old snapshot files that are no longer needed (free up S3 space)
VACUUM orders;
```

**Key facts:**
- `OPTIMIZE` compacts small Parquet files — run periodically after many small writes (streaming ingestion)
- `VACUUM` removes unreferenced snapshot files — run after OPTIMIZE or after large deletes
- Both commands run directly in Athena SQL — no Spark cluster needed

### Exam Patterns
- "Apply GDPR right-to-erasure (delete specific rows) on a data lake using Athena SQL" → Iceberg `DELETE FROM`
- "Upsert CDC records from DMS into a data lake table using Athena" → Iceberg `MERGE INTO`
- "Compact small files in an Iceberg table without Spark" → Athena `OPTIMIZE ... REWRITE DATA`
- "Query the state of a data lake table from 30 days ago" → Iceberg time travel `FOR TIMESTAMP AS OF`
- "Athena vs Glue for Iceberg row-level deletes — which requires less infrastructure?" → Athena (serverless SQL, no Spark cluster needed)

---

## Performance Tuning

### Data Layout Best Practices

| Technique | Impact |
|-----------|--------|
| Parquet / ORC columnar format | Read only queried columns — major scan reduction |
| Snappy compression | Fast decompression, good compression ratio |
| Partition by query dimensions | Skip entire date/region partitions |
| File size 128 MB – 1 GB | Optimal parallelism per file |
| Bucketing | Co-locate data by join key — reduces shuffle in Spark |

### Columnar Format Scan Savings

```
CSV table: 1 TB, SELECT 3 of 100 columns
→ Athena scans 1 TB → costs $5.00

Parquet table (same data): 200 GB (compressed), SELECT 3 columns
→ Athena scans ~6 GB → costs $0.03
```

### Avoid These Anti-patterns

- Querying without partition filters (full table scan)
- Too many small files (< 32 MB) — increases S3 LIST overhead
- Using `SELECT *` on wide tables with many unused columns
- Querying CSV or uncompressed JSON at scale

### Concurrent Query Limits

- Default: 20 concurrent DDL queries, 20 concurrent DML queries per account
- Can be increased via Service Quotas
- Per-workgroup concurrency is not configurable separately — applies at account level

---

## Common Patterns

### Pattern 1: Serverless Data Lake Query

```
S3 (Parquet, partitioned by date) → Glue Catalog → Athena (partition projection)
```
Zero infrastructure, zero maintenance, pay only when queried.

### Pattern 2: CloudTrail / VPC Flow Log Analysis

```
CloudTrail → S3 → Athena table (built-in SerDe) → SQL forensic queries
```
Athena has native SerDes for CloudTrail, VPC Flow Logs, ELB access logs, and CloudFront logs.

### Pattern 3: ETL with CTAS

```
Raw CSV in S3 → Athena CTAS → Parquet + partitioned table in S3
                             → Feed to QuickSight SPICE or Redshift Spectrum
```

### Pattern 4: Federated Join (Enrichment)

```
S3 events table JOIN DynamoDB product table → Athena Federated Query
                                            → Enriched results to S3
```

### Pattern 5: Cost-Controlled Self-Service Analytics

```
Athena Workgroup (per team):
  - Bytes scanned limit per query = 10 GB
  - Result reuse = 60 minutes
  - Engine v3
→ Teams query freely without risk of runaway costs
```

---

## Exam Gotchas

- **"Serverless SQL on S3"** = Athena (always)
- **"Ad-hoc queries" / "infrequent analysis"** = Athena (pay-per-query, no cluster cost)
- **"Optimize Athena cost"** = Parquet + Partitioning + Compression + Result Reuse
- **Partition Projection** = no Glue Crawler needed when partition scheme is predictable
- **Workgroup byte limits** = prevent runaway query costs per team
- **Result reuse** = zero scan cost for repeated identical queries
- **Engine v3 (Trino)** = default for new workgroups, better SQL support than v2
- **CTAS** = one-time format conversion or pre-materialized aggregations
- **UNLOAD** = write query results to S3 without creating a catalog table
- Athena is NOT ideal for complex multi-join OLAP workloads at scale (use Redshift)
- Athena **cannot update or delete rows in plain S3 Parquet** — use CTAS to overwrite or create an Iceberg table for row-level DML
- **Athena Iceberg DML (UPDATE, DELETE, MERGE INTO)** works natively in SQL — no Spark or EMR needed
- **`OPTIMIZE ... REWRITE DATA`** compacts small Iceberg files; **`VACUUM`** removes old snapshots — both are Athena SQL commands
- Federated Query data passes through Lambda — not for bulk external reads
- Cancelled queries are **charged for data already scanned**

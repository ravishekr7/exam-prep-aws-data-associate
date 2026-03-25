# Amazon Redshift

Petabyte-scale SQL data warehouse. OLAP (Online Analytical Processing).

---

## Architecture

### Node Types
| Type | Description | Use Case |
|------|-------------|----------|
| **RA3 (Managed Storage)** | Separate compute and storage. S3 backend | Standard choice. Scale storage independently |
| **DC2 (Dense Compute)** | Fixed SSD storage on node | Smaller datasets where data fits on local disk |

### Components
- **Leader Node:** Handles connections, query parsing, planning, and aggregating results
- **Compute Nodes:** Execute queries in parallel
- **Slices:** Each compute node has slices, each processes a portion of data

---

## Redshift Spectrum

Query data directly in **S3** without loading into Redshift tables.

- Creates "External Tables" registered in Glue Data Catalog
- Join S3 data with Redshift local data
- Runs on Spectrum's own compute fleet (doesn't use your cluster's resources)
- Use case: Offload rarely-accessed historical data to S3

---

## Redshift Federated Queries

Query data in **RDS** or **Aurora** directly from Redshift without ETL.

- Creates external schema pointing to the remote database
- Useful for combining operational and analytical data
- Data stays in source - no copying needed

---

## Materialized Views

- Pre-compute complex query results for faster access
- Need to be refreshed when underlying data changes
- `AUTO REFRESH` option available
- Significant performance boost for repeated analytical queries

---

## Performance Tuning

### Distribution Styles
How rows are distributed across compute nodes:

| Style | Description | Best For |
|-------|-------------|----------|
| **KEY** | Rows with same key go to same node | Tables frequently joined on that key (minimizes shuffle) |
| **ALL** | Full copy on every node | Small dimension tables (< 3M rows) |
| **EVEN** | Round-robin distribution | No obvious join key, uniform distribution |
| **AUTO** | Redshift decides (starts ALL, moves to EVEN/KEY) | Default, let Redshift optimize |

### Sort Keys
- Determines physical order of data on disk
- **Zone Maps:** Redshift skips blocks that don't match the sort key range
- Choose sort key based on WHERE clause and JOIN conditions
- **Compound Sort Key:** Multiple columns, order matters
- **Interleaved Sort Key:** Equal weight to all columns (good for multiple query patterns)

### Workload Management (WLM)
- Create queues to prioritize queries
- **Short Query Acceleration (SQA):** Prioritizes small dashboard queries over long ETL runs
- Set memory allocation per queue
- Concurrency scaling: Add transient clusters for burst query demand

---

## Distribution Style Decision Guide

Choosing the wrong distribution style is the #1 performance killer in Redshift. Here is the decision logic:

### KEY Distribution
**Use when:** Two large tables are frequently JOINed on the same column. By distributing both tables on the join column, matching rows land on the same node — eliminating cross-node data movement (shuffle).

```sql
-- Fact table: distribute on customer_id (join column)
CREATE TABLE fact_orders (
    order_id BIGINT,
    customer_id INT,
    amount DECIMAL(10,2)
) DISTKEY(customer_id);

-- Dimension table: also distribute on customer_id
CREATE TABLE dim_customers (
    customer_id INT,
    customer_name VARCHAR(100)
) DISTKEY(customer_id);

-- This join now has zero data movement (co-located)
SELECT c.customer_name, SUM(o.amount)
FROM fact_orders o JOIN dim_customers c ON o.customer_id = c.customer_id
GROUP BY 1;
```

**Rule:** Both tables in the JOIN must use KEY distribution on the **same join column**. If only one does, you get no benefit.

### ALL Distribution
**Use when:** The table is small (fewer than a few million rows) and appears in many JOIN queries. A full copy on every node means joins to this table never require data movement.

**Never use ALL for large tables** — the data is replicated on every compute node, multiplying storage cost and write time.

### EVEN Distribution
**Use when:** The table is large but has no obvious join partner or varied join patterns. Round-robin ensures balanced storage and avoids skew, but queries that join this table will require data redistribution.

### AUTO Distribution
**Use as starting point.** Redshift starts with ALL (for small tables) and automatically switches to EVEN or KEY as the table grows, based on usage patterns. Monitor with `SVV_TABLE_INFO` to see what Redshift chose.

### Decision Rule Summary
```
Is the table small (< few million rows)?
  YES → ALL (or AUTO, which will likely choose ALL)
  NO → Does it join frequently with another large table on a specific column?
         YES → KEY on that join column (both tables!)
         NO → EVEN (or AUTO)
```

---

## Sort Key Decision Guide

Sort keys determine the physical order of data on disk. Zone maps (min/max metadata per 1 MB block) allow Redshift to skip entire blocks that don't match a WHERE clause filter.

### Compound Sort Key
Columns are listed in priority order. The **first column is the most important**.

```sql
CREATE TABLE fact_sales (
    sale_date DATE,
    region VARCHAR(50),
    product_id INT,
    revenue DECIMAL(10,2)
) COMPOUND SORTKEY(sale_date, region);
-- Excellent for: WHERE sale_date BETWEEN ... AND ...
-- Good for: WHERE sale_date = ... AND region = ...
-- Poor for: WHERE region = ... (skips the first sort column)
```

**Use compound sort key when:**
- Queries almost always filter on the first column (e.g., time-series data filtered by date)
- You have a clear primary filter column

### Interleaved Sort Key
Equal weight given to each column — any column can be the primary filter.

```sql
CREATE TABLE fact_sales (
    sale_date DATE,
    region VARCHAR(50),
    product_id INT
) INTERLEAVED SORTKEY(sale_date, region, product_id);
-- Good for ad-hoc queries that filter on any combination
```

**Downside:** VACUUM REINDEX is very expensive and takes much longer than standard VACUUM. Only use interleaved if query patterns are truly unpredictable across many columns.

### Decision Rule
```
Do queries almost always filter on one or two specific columns in a consistent order?
  YES → Compound sort key (put most-filtered column first)
  NO → Are query filter patterns truly unpredictable across many columns?
         YES → Interleaved sort key (accept expensive VACUUM cost)
         NO → Compound sort key on the most common filter column
```

**For time-series data:** Always use compound sort key on the timestamp/date column. This is the single most impactful optimization for event data.

---

## WLM (Workload Management)

WLM controls how Redshift allocates memory and concurrency slots across different types of queries.

### Manual WLM
You define queues with fixed concurrency slots and memory percentages.
- Risk: if you allocate 80% memory to the ETL queue and 20% to the reporting queue, but both are heavily used simultaneously, one queue starves the other.
- Requires careful tuning and ongoing adjustment.

### Auto WLM (Recommended)
Redshift dynamically adjusts memory allocation and concurrency based on actual query patterns. No fixed slots — Redshift can run more simple queries concurrently and fewer complex queries.

### Short Query Acceleration (SQA)
Redshift ML automatically estimates query execution time. Queries estimated to complete in less than N seconds (configurable) are routed to a **fast lane** that bypasses the normal WLM queue. Simple dashboard queries (SELECT COUNT, simple aggregations) skip ahead of long-running ETL queries.

**Exam pattern:** "Dashboard users complain their queries wait in queue behind ETL jobs" → Enable SQA (routes fast queries to a dedicated lane) or enable Auto WLM.

### Concurrency Scaling
Adds **burst read capacity** to handle peak query demand. When your main cluster is at capacity, Redshift automatically spins up additional transient cluster(s) for read queries. You get 1 hour free per day; charged per second beyond that.

**Exam pattern:** "Queries queue during peak business hours despite adequate cluster size" → Enable Concurrency Scaling on the affected WLM queue.

---

## Redshift Debugging

### System Tables and Views

| Table/View | Purpose | Common Use |
|-----------|---------|-----------|
| `STL_LOAD_ERRORS` | Details of COPY command failures | Diagnose column type mismatches, encoding errors, delimiter issues |
| `STL_QUERY` | Historical record of all queries | Find slow queries by CPU time, row count, execution time |
| `SVL_QUERY_REPORT` | Per-slice execution details for a query | Identify data skew — if one slice processes 10x more rows than others, distribution style is wrong |
| `SVV_TABLE_INFO` | Table statistics, skew, unsorted rows % | Decide when to run VACUUM and ANALYZE; spot tables with high skew or unsorted percentage |

### Debugging Workflow
```sql
-- 1. Check for recent COPY failures:
SELECT * FROM STL_LOAD_ERRORS ORDER BY starttime DESC LIMIT 10;

-- 2. Find slowest recent queries:
SELECT query, datediff(seconds, starttime, endtime) AS duration_sec, label
FROM STL_QUERY
WHERE starttime > DATEADD(hour, -1, GETDATE())
ORDER BY duration_sec DESC
LIMIT 10;

-- 3. Check if a specific query had skew across slices:
SELECT query, slice, rows, bytes
FROM SVL_QUERY_REPORT
WHERE query = <query_id>
ORDER BY slice;

-- 4. Check tables that need maintenance:
SELECT "table", size, skew_rows, unsorted, stats_off
FROM SVV_TABLE_INFO
WHERE skew_rows > 1.5 OR unsorted > 20 OR stats_off > 10
ORDER BY size DESC;
```

---

## Data Loading

### COPY Command (Preferred)
- Load from S3, DynamoDB, EMR, remote hosts
- Parallel loading from multiple files = fastest
- Supports Parquet, ORC, CSV, JSON
- Use MANIFEST file to specify exact files

### UNLOAD Command
- Export Redshift query results to S3
- Supports Parquet format
- Use for archiving or feeding data to other services

---

## Redshift Serverless

- No cluster management
- Auto-scales compute based on workload
- Pay for compute used (RPU - Redshift Processing Units)
- Good for intermittent or unpredictable query patterns

---

## Exam Gotchas

- **COPY command** is always preferred over INSERT for bulk loading
- **Distribution KEY** = answer when "optimize joins" or "minimize data shuffling"
- **ALL distribution** = small dimension tables only
- **Spectrum** runs on its own compute fleet - offloads work from your cluster
- **Vacuum** reclaims space from deleted rows and re-sorts data (auto-vacuum runs in background)
- **ANALYZE COMPRESSION:** Recommends optimal column compression encodings
- Redshift writes to Redshift via intermediate S3 (Firehose -> S3 -> COPY)
- **Concurrency Scaling** = handle burst query demand without resizing cluster
- **Upsert pattern:** Load to Staging -> Delete matching from Target -> Insert from Staging (or use MERGE)
- **COPY command is the fastest way to load data from S3** — always choose COPY over INSERT for bulk loads. INSERT row by row is extremely slow in Redshift.
- **UNLOAD exports in parallel** to multiple S3 files — fast for large exports. Use `PARALLEL OFF` only for small exports where a single file is required.
- **Redshift Spectrum** lets you query S3 data as if it's in Redshift tables. Data stays in S3 — no loading needed. Spectrum is ideal for infrequently queried historical (cold) data that would be expensive to store in Redshift.
- **Materialized views with AUTO REFRESH** pre-compute expensive joins and aggregations. BI dashboards that always run the same complex query should use materialized views — dramatically reduces query time and cluster load.
- **RA3 nodes** decouple compute from storage. Data is stored in S3-based **Redshift Managed Storage (RMS)**, with frequently accessed data cached on local NVMe SSDs. You pay for compute and storage separately, so you can scale each independently.

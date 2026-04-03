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

## Redshift Streaming Ingestion

Direct ingestion from **Kinesis Data Streams** or **Amazon MSK (Kafka)** into Redshift via a materialized view — without Firehose, S3, or a COPY command.

**How It Works:**
1. Create an external schema in Redshift pointing to the Kinesis stream or MSK topic
2. Create a materialized view that selects from the external schema (the stream)
3. Refresh the materialized view to pull new records into Redshift

```sql
-- Step 1: Create external schema for a Kinesis stream
CREATE EXTERNAL SCHEMA kinesis_schema
FROM KINESIS
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftKinesisRole';

-- Step 2: Create materialized view over the stream
CREATE MATERIALIZED VIEW orders_stream AS
SELECT approximatearrivaltimestamp,
       JSON_PARSE(from_varbyte(data, 'utf-8')) AS order_data
FROM kinesis_schema.order_events_stream;

-- Step 3: Refresh to pull new records
REFRESH MATERIALIZED VIEW orders_stream;
-- Can schedule this with Redshift Scheduler or EventBridge
```

**Why It Matters — Firehose vs Streaming Ingestion:**

| Approach | Latency | Complexity | Best For |
|---|---|---|---|
| Firehose → S3 → COPY | 60 seconds minimum (buffer) | Medium | Durable, cost-optimized batch loads |
| Redshift Streaming Ingestion | Sub-second | Low (no intermediate store) | Near-real-time analytics on fresh data |

**Key Constraints:**
- Data consumed from the stream is raw bytes — use `JSON_PARSE` or `FROM_VARBYTE` to deserialize
- Supports Kinesis Data Streams and Amazon MSK; does not support Kinesis Firehose as the source
- Requires an IAM role with `kinesis:GetRecords`, `kinesis:GetShardIterator`, `kinesis:DescribeStream`

**Exam Patterns:**
- "Ingest streaming Kinesis data into Redshift with sub-second latency and least operational overhead" → Redshift Streaming Ingestion (not Firehose, which buffers to S3 first)
- "Real-time dashboards on Redshift need data within seconds of it arriving in Kinesis" → Redshift Streaming Ingestion

---

## Materialized Views

- Pre-compute complex query results for faster access
- Need to be refreshed when underlying data changes
- `AUTO REFRESH` option available
- Significant performance boost for repeated analytical queries

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

> **Deprecated.** AWS deprecated interleaved sort keys. New tables should use compound sort keys. Existing tables with interleaved sort keys continue to work but AWS no longer recommends creating new ones.

Equal weight given to each column — any column can be the primary filter.

**Why it was deprecated:** `VACUUM REINDEX` (required to maintain interleaved sort key effectiveness after large data loads) is extremely expensive and slow — often taking longer than the data load itself. AWS found compound sort keys with careful column ordering outperform interleaved in practice.

**Exam implication:** If a question asks which sort key to use for unpredictable, multi-column filter patterns, the answer is still compound sort key on the most commonly filtered column — not interleaved. Interleaved is a distractor.

### Decision Rule
```
Do queries almost always filter on one or two specific columns in a consistent order?
  YES → Compound sort key (put most-filtered column first)
  NO → Are query filter patterns truly unpredictable across many columns?
         YES → Compound sort key on most common filter column (interleaved is deprecated)
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
Redshift automatically estimates query execution time using an internal WLM predictor. Queries estimated to complete in less than N seconds (configurable) are routed to a **fast lane** that bypasses the normal WLM queue. Simple dashboard queries (SELECT COUNT, simple aggregations) skip ahead of long-running ETL queries.

Note: SQA is part of WLM and is unrelated to **Redshift ML** (the separate feature for training ML models using `CREATE MODEL` SQL syntax).

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
- **Requires an IAM role attached to the Redshift cluster** to access S3 — Redshift uses the role's permissions, not user credentials

```sql
-- COPY using an IAM role attached to the cluster
COPY fact_sales
FROM 's3://my-bucket/data/sales/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3Role'
FORMAT AS PARQUET;
```

Exam pattern: "Grant Redshift permission to load data from S3" → Attach an IAM role with `s3:GetObject` permissions to the cluster and reference it in the COPY command.

### UNLOAD Command
- Export Redshift query results to S3
- Supports Parquet format
- Use for archiving or feeding data to other services

---

## Enhanced VPC Routing

By default, COPY and UNLOAD traffic between Redshift and S3 routes over the public internet — even if both are in the same AWS region.

**Enhanced VPC Routing** forces all COPY/UNLOAD traffic to travel through your VPC, using VPC endpoints (S3 gateway endpoint) rather than the public internet.

**Why it matters:**
- Without Enhanced VPC Routing: Redshift → S3 traffic may exit your VPC via the public internet
- With Enhanced VPC Routing: traffic stays within the AWS network, routable through VPC endpoints, VPC peering, or Direct Connect

**Prerequisites when enabling:**
- An S3 VPC gateway endpoint must be configured in the VPC — otherwise COPY/UNLOAD will fail because there is no private path to S3
- NAT gateway or internet gateway can serve as fallback if the VPC endpoint is missing, but this defeats the purpose of Enhanced VPC Routing

**Exam Patterns:**
- "Ensure all data traffic between Redshift and S3 stays within the AWS network and never traverses the public internet" → Enable Enhanced VPC Routing + configure S3 VPC gateway endpoint
- "COPY commands started failing after enabling Enhanced VPC Routing" → Missing S3 VPC gateway endpoint in the subnet route table

---

## Redshift ML

Create, train, and invoke ML models using standard SQL — without moving data out of Redshift. Under the hood, Redshift exports training data to S3 and calls **Amazon SageMaker AutoPilot** to select and train the best model. Once trained, a prediction function is registered directly in Redshift and callable from SQL.

```sql
-- Step 1: Train a model (Redshift exports data to S3, SageMaker trains it)
CREATE MODEL predict_customer_churn
FROM (
    SELECT age, total_orders, days_since_last_order, churn
    FROM customers
    WHERE split = 'train'
)
TARGET churn
FUNCTION ml_churn_predict
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftMLRole'
SETTINGS (S3_BUCKET 's3://my-redshift-ml-artifacts/');

-- Step 2: Check model status (training happens asynchronously in SageMaker)
SHOW MODEL predict_customer_churn;

-- Step 3: Run inference in SQL — no API calls, no Python
SELECT customer_id,
       ml_churn_predict(age, total_orders, days_since_last_order) AS churn_probability
FROM customers
WHERE split = 'inference';
```

**Key Facts:**
- Training is done in SageMaker (outside Redshift) — the IAM role needs permissions for both Redshift and SageMaker
- `AUTO OFF` option: bring your own SageMaker model ARN instead of using AutoPilot
- Inference (prediction calls) runs **inside Redshift** — fast, no data leaves the warehouse, no extra cost per call
- `CREATE MODEL` is asynchronous — the model status goes from `TRAINING` → `READY`

**Exam Patterns:**
- "Data analysts want to run ML predictions in SQL without learning Python or moving data to SageMaker" → Redshift ML
- "Train a classification model on existing Redshift data with least operational overhead" → `CREATE MODEL` with AutoPilot
- **Note:** SQA (Short Query Acceleration) is a WLM feature — it is NOT related to ML or Redshift ML. The exam will offer both as distractors.

---

## Redshift Serverless

- No cluster management
- Auto-scales compute based on workload
- Pay for compute used (RPU - Redshift Processing Units)
- Good for intermittent or unpredictable query patterns

---

## Redshift Data Sharing

Redshift Data Sharing allows live data sharing between Redshift clusters or Redshift Serverless namespaces — in the same account or across accounts — without copying or moving any data.

**The Problem It Solves:**

Before Data Sharing: Team A and Team B each have their own Redshift clusters. To share data, one team must run an UNLOAD to S3, the other runs a COPY. Data is stale, duplicated, and requires a manual pipeline to keep in sync.

With Data Sharing: Team A creates a datashare (a named object pointing to specific databases, schemas, or tables). Team B mounts it as a consumer cluster and queries it in real time — always seeing Team A's current data with no ETL.

**Key Requirement:** Both producer and consumer must use **RA3 nodes** (or be Redshift Serverless). DC2 clusters cannot participate in Data Sharing — this is a hard constraint. A question describing a DC2 cluster asking if Data Sharing is feasible: the answer is no without migrating to RA3.

**Key Concepts:**

| Concept | Description |
|---|---|
| Datashare | Named collection of database objects (tables, views, schemas) made available for sharing |
| Producer cluster | The cluster that owns the data and creates the datashare |
| Consumer cluster | The cluster that mounts and queries the datashare |
| Read-only | Consumer can only SELECT from shared objects — cannot INSERT, UPDATE, DELETE |

**Same-account sharing:**
```sql
-- On producer cluster: create a datashare
CREATE DATASHARE salesshare;

-- Add objects to the datashare
ALTER DATASHARE salesshare ADD SCHEMA public;
ALTER DATASHARE salesshare ADD TABLE public.fact_sales;

-- Grant access to consumer cluster
GRANT USAGE ON DATASHARE salesshare TO NAMESPACE 'consumer-cluster-namespace-id';

-- On consumer cluster: mount the datashare as a database
CREATE DATABASE sales_from_producer FROM DATASHARE salesshare OF NAMESPACE 'producer-namespace-id';

-- Query it directly
SELECT * FROM sales_from_producer.public.fact_sales LIMIT 10;
```

**Cross-account sharing:**

Same as above but uses AWS Data Exchange or direct account ID to share the datashare. The consumer account's Redshift admin must accept the share before it can be mounted.

**Exam Patterns:**
- "Two teams use separate Redshift clusters but need to query each other's data in real-time without ETL" → Redshift Data Sharing
- "Analytics team needs read access to the transactional team's Redshift data without impacting their cluster's performance" → Data Sharing (consumer queries run on consumer cluster compute, not producer)
- "Share live Redshift data across AWS accounts" → Cross-account Redshift Data Sharing (vs Redshift Spectrum which queries S3, not other Redshift clusters)

**Data Sharing vs Redshift Spectrum vs Federated Queries:**

| Feature | Data Sharing | Redshift Spectrum | Federated Queries |
|---|---|---|---|
| Data source | Another Redshift cluster | S3 (external tables) | RDS / Aurora |
| Data is live? | Yes (real-time) | Yes (reads S3 directly) | Yes (reads source DB) |
| Copy needed? | No | No | No |
| Use case | Redshift-to-Redshift sharing | Analytics on S3 cold data | Joining Redshift with operational DB |

---

## Resize Operations — Elastic vs Classic

When your Redshift cluster needs more or fewer nodes, there are two resize methods. Choosing the wrong one causes unnecessary downtime.

| Feature | Elastic Resize | Classic Resize |
|---|---|---|
| Completion time | Minutes (2–10 minutes typical) | Hours (can take many hours for large clusters) |
| Downtime | Brief read-only period during resize (cluster stays up, writes paused momentarily) | Cluster is unavailable for the entire resize duration |
| Node type change | Cannot change node type (only node count) | Can change both node type AND node count |
| Data redistribution | Happens in the background after resize completes | Happens during resize (blocking) |
| Use case | Scaling node COUNT up or down on the same node type | Migrating from dc2 → ra3, or changing instance family |

**When to use Elastic Resize:**
- Adding or removing nodes of the same type (e.g., ra3.xlplus: 2 → 4 nodes)
- You need to complete the resize with minimal impact
- "Least downtime" or "minimal disruption" in the question → Elastic Resize

**When to use Classic Resize:**
- Migrating from an older node type to RA3 (e.g., dc2.large → ra3.xlplus)
- Changing both node type AND count simultaneously
- Scenario describes a major cluster overhaul during a maintenance window

**Snapshot and restore (alternative to Classic Resize):**

For very large clusters where Classic Resize would take many hours, the alternative is:
1. Take a snapshot of the cluster
2. Restore the snapshot to a new cluster with the desired node type/count
3. UNLOAD data from old cluster and COPY to new cluster (for any data written during migration)
4. Redirect applications to new cluster
5. Terminate old cluster

This approach minimizes actual production downtime at the cost of more manual steps.

**Exam Patterns:**
- "Scale a Redshift cluster from 4 to 8 ra3.xlplus nodes with minimum downtime" → Elastic Resize
- "Migrate a Redshift cluster from dc2.large nodes to ra3.xlplus nodes" → Classic Resize (node type change requires Classic)
- "Resize a Redshift cluster with zero write downtime" → Not possible with Classic Resize; use Elastic Resize (which has only a brief read-only period, not full unavailability)

---

## Exam Gotchas

- **COPY command** is always preferred over INSERT for bulk loading
- **Distribution KEY** = answer when "optimize joins" or "minimize data shuffling"
- **ALL distribution** = small dimension tables only
- **Spectrum** runs on its own compute fleet - offloads work from your cluster
- **Vacuum** reclaims space from deleted rows and re-sorts data (auto-vacuum runs in background)
- **ANALYZE COMPRESSION:** Recommends optimal column compression encodings
- **Kinesis Firehose loads into Redshift via intermediate S3** (Firehose → S3 → COPY). Firehose cannot write to Redshift directly — it buffers to S3 first, then issues a COPY command
- **Concurrency Scaling** = handle burst query demand without resizing cluster
- **Upsert pattern:** Load to Staging -> Delete matching from Target -> Insert from Staging (or use MERGE)
- **COPY command is the fastest way to load data from S3** — always choose COPY over INSERT for bulk loads. INSERT row by row is extremely slow in Redshift.
- **UNLOAD exports in parallel** to multiple S3 files — fast for large exports. Use `PARALLEL OFF` only for small exports where a single file is required.
- **Redshift Spectrum** lets you query S3 data as if it's in Redshift tables. Data stays in S3 — no loading needed. Spectrum is ideal for infrequently queried historical (cold) data that would be expensive to store in Redshift.
- **Materialized views with AUTO REFRESH** pre-compute expensive joins and aggregations. BI dashboards that always run the same complex query should use materialized views — dramatically reduces query time and cluster load.
- **RA3 nodes** decouple compute from storage. Data is stored in S3-based **Redshift Managed Storage (RMS)**, with frequently accessed data cached on local NVMe SSDs. You pay for compute and storage separately, so you can scale each independently.
- **Redshift Data Sharing ≠ Redshift Spectrum.** Data Sharing is Redshift-to-Redshift live data access. Spectrum queries S3 external tables. They solve different problems — the exam will try to confuse you by offering both as options.
- **Elastic Resize only changes node count, not node type.** If the scenario requires changing the instance family (e.g., DC2 to RA3), Elastic Resize is not an option — you must use Classic Resize or the snapshot-restore approach.
- **Consumer cluster in Data Sharing uses its OWN compute** for queries. Sharing data does not impact the producer cluster's query performance — consumers pay for their own query execution.
- **Redshift Streaming Ingestion ≠ Kinesis Firehose.** Streaming Ingestion is direct (sub-second latency, no S3 buffer). Firehose buffers to S3 first (minimum 60-second delay). Use Streaming Ingestion when the question asks for "sub-second" or "real-time" Redshift ingestion.
- **Redshift ML uses SageMaker AutoPilot under the hood** — training happens outside Redshift. The IAM role needs both Redshift and SageMaker permissions, plus access to the S3 bucket for artifacts. SQA is NOT related to ML; it is a WLM routing feature for short queries.

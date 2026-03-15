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

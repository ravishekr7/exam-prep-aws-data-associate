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

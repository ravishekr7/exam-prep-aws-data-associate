# Performance Tuning

Optimizing Spark, Redshift, and Kinesis for data engineering workloads.

---

## Apache Spark (Glue/EMR)

### Data Skew
- One partition has 90% of data (e.g., `country=US`)
- One executor works forever, others finish instantly
- **Fixes:** Salt the key, choose better partition key, `repartition()`

### Small Files Problem
- Too many small files (KB-size) = excessive S3 LIST overhead
- Metadata operations become bottleneck
- **Fixes:**
  - `groupFiles` option in Glue
  - `coalesce()` / `repartition()` before writing
  - Firehose buffering (time/size)
  - Post-hoc compaction job

### Optimal File Size
- Sweet spot: 128 MB - 1 GB per file
- Too small = metadata overhead
- Too large = can't parallelize effectively

### Spark Tuning Tips
- **Broadcast joins:** For small tables (< 10 MB), broadcast to all executors
- **Cache/Persist:** Reuse DataFrames that are accessed multiple times
- **Avoid shuffles:** Minimize `groupByKey`, prefer `reduceByKey`
- **Right-size executors:** Balance memory and cores

---

## Redshift Performance

### Distribution Keys
- Wrong key = "Broadcasting" (shuffling entire table across network)
- **KEY distribution** on frequently joined columns
- **ALL distribution** for small dimension tables

### Sort Keys
- Enables zone map pruning (skip irrelevant blocks)
- Choose columns used in WHERE and ORDER BY clauses

### Compression
- `ANALYZE COMPRESSION` recommends optimal encodings
- Reduces I/O and storage

### Vacuum and Analyze
- **VACUUM:** Reclaims space from deleted rows, re-sorts data
- **ANALYZE:** Updates statistics for query optimizer
- Auto-vacuum runs in background, but manual may be needed after large loads

### Concurrency Scaling
- Adds transient clusters for burst query demand
- Queries routed to scaling cluster when main cluster is busy
- Free credits: 1 hour per 24 hours of main cluster usage

---

## Kinesis Performance

### Iterator Age
- CloudWatch metric showing how far behind the consumer is
- High iterator age = processing lag
- **Fixes:** Increase shard count, optimize consumer logic, increase Lambda batch size

### ProvisionedThroughputExceeded
- Hitting the shard write/read limit
- **Fixes:** Split shards (resharding), switch to On-Demand mode, use Enhanced Fan-Out for consumers

### Enhanced Fan-Out
- Dedicated 2 MB/sec per consumer (not shared)
- Use when multiple consumers need low-latency access

---

## Backup and Disaster Recovery

### RPO vs RTO
- **RPO (Recovery Point Objective):** How much data can you lose? (time)
- **RTO (Recovery Time Objective):** How fast must you recover? (time)

### Service-Specific DR
| Service | Strategy |
|---------|----------|
| **RDS/Redshift** | Automated snapshots (1-35 days). Cross-region snapshot copy |
| **S3** | Cross-Region Replication (CRR). Versioning for accidental delete protection |
| **DynamoDB** | PITR (last 35 days, per-second granularity). Global Tables (active-active) |
| **Aurora** | Global Database (< 1s cross-region replication) |

---

## Exam Gotchas

- **"Data skew"** = salting or repartitioning
- **"Small files"** = groupFiles, coalesce, or Firehose buffering
- **"Kinesis lag"** = check IteratorAge metric, add shards
- **"Redshift slow queries"** = check distribution key, sort key, VACUUM
- **Concurrency Scaling** = handle burst query demand without resizing Redshift cluster
- RPO/RTO questions: match the DR strategy to the acceptable data loss and recovery time

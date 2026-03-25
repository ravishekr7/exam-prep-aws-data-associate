# Amazon EMR (Elastic MapReduce)

Big data platform for running Apache Spark, Hive, Presto, HBase, and other distributed frameworks on AWS.

---

## Architecture

### Cluster Node Types
| Node | Role |
|------|------|
| **Primary (Master)** | Manages cluster, runs YARN ResourceManager, HDFS NameNode |
| **Core** | Runs tasks AND stores data (HDFS). Removing core nodes = data loss |
| **Task** | Runs tasks only, no HDFS. Safe to add/remove. Great for Spot instances |

### Deployment Modes
- **EMR on EC2:** Traditional cluster with full control over instances
- **EMR on EKS:** Run Spark on Kubernetes
- **EMR Serverless:** No infrastructure management, auto-scales, pay per use

---

## EMR Deployment Mode Comparison

| Dimension | EMR on EC2 | EMR Serverless | EMR on EKS |
|-----------|------------|----------------|------------|
| **Management overhead** | High — you manage cluster lifecycle, node types, scaling policies | None — fully managed | Medium — EKS cluster managed separately, EMR on EKS handles Spark |
| **Cost model** | Per EC2 instance-hour. Spot instances available for 60-90% savings | Pay per vCPU-second and memory-second used (per job). No idle cost. | Per EC2 instance-hour (EKS node groups) |
| **When to choose** | Sustained heavy workloads where Spot savings justify management; need specific JARs, OS config, or frameworks beyond Spark (Hive, Presto, HBase) | Variable/bursty workloads; simplicity is priority; don't want to manage clusters | Already operating EKS; want unified compute plane for containers + Spark; multi-tenant environments |
| **Spark version support** | Full control — specify any supported version | Latest stable versions, AWS-managed | Latest stable versions, AWS-managed |
| **Auto-scaling** | Manual or YARN-based auto-scaling policies | Fully automatic (0 to N workers per job) | Kubernetes-based auto-scaling via Cluster Autoscaler |
| **Cold start** | Minutes (cluster launch) | 30-60 seconds (worker provisioning per job) | Seconds if nodes already running in node group |
| **Spot support** | Yes — core and task nodes | Pre-initialized capacity available (optional) | Yes — via managed node groups |

### Rule of Thumb
- **EMR on EC2 with Spot** = cheapest for sustained, high-volume workloads running many hours/day
- **EMR Serverless** = easiest operational experience for variable/bursty batch jobs
- **EMR on EKS** = when your organization already runs EKS and wants unified compute for containers and Spark

---

## Storage Options

| Option | Description |
|--------|-------------|
| **HDFS** | Local cluster storage, fast but ephemeral (lost when cluster terminates) |
| **EMRFS (S3)** | Persistent storage via S3. Recommended for data lakes. Supports consistent reads |
| **EBS** | Attached volumes for temporary storage |

---

## EMRFS

EMRFS (EMR File System) provides an HDFS-compatible interface to Amazon S3 from within an EMR cluster. Instead of your Spark jobs needing to use S3 URLs directly, EMRFS handles the translation transparently.

### Key Points
- **S3 strong consistency (since December 2020):** AWS made S3 strongly consistent for all GET, PUT, LIST operations. The old EMRFS "consistent view" feature (which used DynamoDB to track S3 object state) is **no longer required** for most use cases.
- **EMRFS S3-optimized committer:** A high-performance output committer for Spark jobs writing Parquet to S3. The standard Spark committer uses expensive rename operations; the S3-optimized committer uses multipart uploads and avoids renames entirely — significantly faster job commit phase.

```
# Enable S3-optimized committer in spark-defaults.conf or job config:
spark.sql.parquet.output.committer.class=com.amazon.emr.committer.EmrOptimizedSparkSqlParquetOutputCommitter
spark.hadoop.mapreduce.outputcommitter.factory.scheme.s3=com.amazon.emr.committer.EmrOptimizedSparkSqlParquetOutputCommitter
```

---

## Key Frameworks

| Framework | Use Case |
|-----------|----------|
| **Apache Spark** | General-purpose batch + streaming processing |
| **Apache Hive** | SQL queries on large datasets (HiveQL) |
| **Presto/Trino** | Interactive SQL queries across multiple sources |
| **HBase** | NoSQL columnar store on HDFS |
| **Apache Flink** | Real-time stream processing |

---

## Spark Performance Tuning

### spark.sql.shuffle.partitions
Default value is **200**, which is almost always wrong.
- Too few partitions → each partition is huge, tasks take long, risk OOM
- Too many partitions → scheduler overhead, many tiny files written to S3

**Target:** ~128 MB per partition after a shuffle.

```
# Formula:
# shuffle_partitions = total_shuffle_data_GB * 1024 / 128

# Example: 100 GB shuffle data
shuffle_partitions = 100 * 1024 / 128 = 800 partitions

# Set in spark-submit or SparkSession:
spark.conf.set("spark.sql.shuffle.partitions", "800")
```

### Executor Sizing: Starting Point
```python
# Typical starting configuration for r5.4xlarge nodes (16 vCPU, 128 GB RAM):
spark.executor.cores = 4          # 4 cores per executor (leave 1 vCPU for OS + YARN)
spark.executor.memory = 16g       # 16 GB per executor
# This gives 3 executors per node (12 cores used + 4 for OS/YARN)

# Memory overhead (off-heap): default 10% of executor memory, min 384 MB
spark.executor.memoryOverhead = 2g
```

### Data Skew
**Symptoms:** Most Spark tasks finish in 2-3 minutes, but 1-2 tasks take 30+ minutes. The slow tasks process disproportionately more data.

**Causes:** Poor partition key (e.g., `country` column where 80% of records are `"US"`).

**Solutions:**
1. **Salting:** Add random suffix to skewed key to spread across partitions
   ```python
   from pyspark.sql.functions import concat, lit, floor, rand
   df = df.withColumn("salted_key", concat(col("country"), lit("_"), (floor(rand() * 10)).cast("string")))
   df = df.repartition("salted_key")
   ```
2. **Broadcast join:** For joins where one table is small, broadcast it to avoid shuffle entirely (see below)
3. **Repartition before join:** Explicitly repartition on join key before joining

### Broadcast Join
When joining a large table with a small table (< ~10 MB), use a broadcast join. Spark sends the small table to every executor, so the join happens locally without any data shuffle.

```python
from pyspark.sql.functions import broadcast

# Spark auto-broadcasts tables smaller than spark.sql.autoBroadcastJoinThreshold (default 10MB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50mb")  # increase if needed

# Or force broadcast explicitly:
result = large_df.join(broadcast(small_df), "join_key")
```

### Things to Avoid
| Anti-Pattern | Why Bad | Alternative |
|-------------|---------|-------------|
| `df.collect()` on large dataset | Pulls all data to driver — OOM risk | Use `df.write` or process in partitions |
| Python UDFs | Serialization overhead; JVM→Python→JVM for each row. 10-100x slower than native Spark functions | Use `pyspark.sql.functions` built-ins or Pandas UDFs (vectorized) |
| `SELECT *` then filter | Reads all columns unnecessarily (Parquet is columnar) | Select only needed columns first |

---

## EMR Instance Fleet vs Instance Group

| Feature | Instance Group | Instance Fleet |
|---------|---------------|----------------|
| **Instance types per group** | 1 fixed type | Up to 5 different types |
| **Spot + On-Demand mix** | One or the other per group | Mix On-Demand and Spot within same fleet |
| **Spot availability** | Single type = higher risk of no availability | Multiple types = AWS picks available Spot pools |
| **Configuration** | Simpler | More complex but more resilient |
| **Scaling** | Group-level scaling policies | Fleet-level with weighted capacity |

### When to Use Each
- **Instance Group:** Simple workloads, predictable instance type, or when you need On-Demand only
- **Instance Fleet:** Production workloads where you want Spot savings AND resilience (if one instance type is unavailable, AWS uses another)

**Exam pattern:** "minimize cost while maintaining availability for EMR task nodes" → **Instance Fleet** with Spot instances across multiple instance types (e.g., `m5.xlarge`, `m5a.xlarge`, `m4.xlarge`).

---

## Cost Optimization

- **Spot Instances:** Use for Task nodes (90% cheaper, fault-tolerant)
- **Reserved Instances:** For Core/Primary nodes with predictable workloads
- **Graviton Instances:** 20% better price-performance
- **Transient Clusters:** Spin up for a job, tear down after. Use S3 (EMRFS) for persistence
- **Auto-scaling:** Scale core/task nodes based on YARN metrics

---

## EMR vs Glue

| Feature | EMR | Glue |
|---------|-----|------|
| **Management** | You manage cluster | Serverless |
| **Control** | Full (OS, JARs, configs) | Limited |
| **Cost at Scale** | Lower (Spot + Reserved) | Higher (DPU pricing) |
| **Startup** | Minutes | 10-30 seconds |
| **Best For** | Constant heavy load, custom tools | Pure ETL, irregular schedules |
| **Frameworks** | Spark, Hive, Presto, HBase, Flink | Spark, Python Shell, Ray |

---

## Exam Gotchas

- Use **Task nodes** for Spot instances (no data loss risk)
- **Transient clusters** with EMRFS/S3 = cost-effective batch processing
- EMR Serverless removes all infrastructure management (like Glue but with more framework options)
- EMR gives better cost at massive, constant-load scale compared to Glue
- HDFS data is **lost** when cluster terminates - always persist important data to S3
- **EMR Bootstrap Actions** run on each node **before** applications (Spark, Hive) are installed. Use bootstrap actions to install custom software, configure OS settings, or pre-download artifacts. They run as root and can execute any shell commands.
- **EMR Step:** A unit of work submitted to the cluster (e.g., a Spark job JAR, a Hive script). Steps are queued and executed sequentially (or in parallel if configured). You can chain steps so that Step 2 only runs if Step 1 succeeds. The cluster can be configured to **auto-terminate** after all steps complete.
- **Transient cluster pattern:** Launch cluster → run steps → auto-terminate. Pay only for the duration of the job. Data persists in S3 (EMRFS). This is the cost-optimal pattern for intermittent batch workloads. Contrast with **persistent cluster** (always running, lower latency but higher cost).

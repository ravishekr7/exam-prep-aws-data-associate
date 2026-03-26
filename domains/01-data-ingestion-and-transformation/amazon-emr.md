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

## EMR Security Configuration

EMR Security Configurations are a cluster-level object defining encryption for data at rest and in transit. They are created once and applied at cluster launch time — they cannot be added to a running cluster.

### Encryption at Rest

| Data Location | Encryption Method | Notes |
|--------------|------------------|-------|
| HDFS (local disk on EC2 Core/Task nodes) | LUKS (Linux Unified Key Setup) via EBS encryption | Applied to EBS volumes attached to each node |
| S3 / EMRFS | SSE-S3, SSE-KMS, or **CSE-KMS** (Client-Side Encryption with KMS) | CSE-KMS: data is encrypted **before** leaving the node, then written to S3. Most secure. |
| Local instance store (NVMe SSD) | LUKS | Applied to instance store volumes on storage-optimised instances |

### Encryption in Transit

TLS encryption for data moving between EMR nodes — Spark shuffle data, HDFS block transfers, inter-node communication.

- Requires a TLS certificate. Provide via: ACM Private CA, a custom Lambda-based certificate provider, or a self-signed cert stored in S3.
- Enable in Security Configuration: set **"In-transit encryption"** and provide the certificate provider ARN or S3 path.

**Exam pattern:** "The security team requires all data moving between EMR cluster nodes to be encrypted" → Enable **in-transit encryption** in EMR Security Configuration with a TLS certificate.

### Authentication — Kerberos

- EMR supports Kerberos to control which users can authenticate to the cluster and submit jobs
- Without Kerberos: any user who can SSH to the cluster can run Spark/Hive jobs
- With Kerberos: users must authenticate with a KDC (Key Distribution Center) before job submission
- EMR can run its own managed KDC or integrate with an external **Active Directory** (via AD Connector)

**Exam pattern:** "Ensure only authenticated corporate users can submit Spark jobs to the EMR cluster" → Enable **Kerberos authentication** in EMR Security Configuration.

### IAM Roles for EMR — Three Distinct Roles (Commonly Confused)

| Role | Who Uses It | Purpose | Example Permissions |
|------|------------|---------|---------------------|
| **EMR Service Role** | The EMR service itself | Allows EMR to provision EC2 instances, VPC resources, and S3 log paths on your behalf | `ec2:RunInstances`, `ec2:TerminateInstances`, `iam:PassRole` |
| **EC2 Instance Profile** | Your Spark/Hive/Presto code running on the nodes | What your jobs use when calling AWS APIs from within the cluster | `s3:GetObject`, `s3:PutObject`, `glue:GetTable`, `dynamodb:PutItem` |
| **Auto Scaling Role** | The Auto Scaling service | Allows EMR to call Auto Scaling APIs to add/remove task nodes | `cloudwatch:PutMetricAlarm`, `autoscaling:SetDesiredCapacity` |

> **Exam trap — "EMR job cannot read from S3":** Check the **EC2 Instance Profile**. The Spark driver and executors use the Instance Profile to authenticate AWS API calls. The EMR Service Role is irrelevant to S3 data access.

> **Exam trap — "EMR cannot launch cluster — insufficient permissions":** Check the **EMR Service Role**. It needs permissions to provision EC2, configure VPC interfaces, and write logs to S3.

**Exam patterns for this section:**
- "Encrypt data at rest on HDFS local disks" → Security Configuration with EBS encryption (LUKS)
- "Encrypt Spark shuffle data between nodes" → in-transit encryption in Security Configuration
- "EMR jobs must use a customer-managed KMS key when writing to S3 — key must not exist in AWS unencrypted" → **CSE-KMS** (client-side encryption — data encrypted before leaving the node, KMS key never exposed on S3 side)
- "Least privilege for EMR nodes reading from S3" → scope the **EC2 Instance Profile** to specific S3 bucket ARNs

---

## Spot Instance Recoverability and Task Node Behaviour

### What Happens When a Spot Task Node Is Reclaimed

1. AWS sends a **2-minute interruption notice** to the instance
2. YARN marks the node as unhealthy and stops scheduling new tasks to it
3. Running Spark tasks on the reclaimed node are marked **FAILED**
4. Spark's built-in fault tolerance **automatically reschedules** those failed tasks on surviving nodes
5. The job continues — no manual intervention required
6. Because Task nodes hold **no HDFS data** (only Core nodes store HDFS blocks), there is **no data loss**

### Why Task Nodes Are Safe for Spot, Core Nodes Are Not

| Node Type | Holds HDFS Data? | Safe for Spot? | Reason |
|-----------|-----------------|----------------|--------|
| **Primary (Master)** | Yes (NameNode metadata) | **No** | Losing primary = cluster failure — entire HDFS namespace is gone |
| **Core** | Yes (HDFS data blocks) | **Risky** | Losing core nodes = HDFS block loss. HDFS decommission is slow (data must be replicated away first). |
| **Task** | No | **Yes** | Pure compute only. Tasks reschedule automatically. Zero data impact. |

### Recommended Cost-Optimised Architecture

```
Primary node:  1 × On-Demand   (cluster would die without it)
Core nodes:    2-3 × On-Demand (minimum for HDFS replication factor 3)
Task nodes:    Spot via Instance Fleet with 3-5 instance types
               (maximize Spot pool availability, handles interruptions gracefully)
```

### Exam Patterns
- "EMR cluster loses several Task nodes during processing. Spark jobs fail completely." → Core nodes were misconfigured as Spot — they held HDFS data. Should have used Task nodes for Spot.
- "Minimise EMR cost for a 10-hour daily batch job while tolerating occasional node loss" → On-Demand Primary + On-Demand Core (minimum) + Spot Task nodes via Instance Fleet
- "A Spot Task node is reclaimed mid-job. Does the job fail?" → **No**. Spark reschedules the affected tasks on remaining nodes. The job slows down temporarily but does not fail (unless ALL nodes are reclaimed simultaneously).

### Spot Handling Best Practices

```bash
# Keep a minimum of On-Demand Core nodes in Instance Fleet:
# Set maximumCoreCapacityUnits in Instance Fleet configuration

# Enable Cluster Managed Scaling (recommended over manual Auto Scaling):
# EMR automatically adds Task nodes when YARN queue backs up,
# removes them when idle — no CloudWatch alarms needed

# Increase Spark task failure tolerance:
spark.task.maxFailures=8   # default is 4; higher value tolerates more Spot interruptions
                           # before Spark aborts the stage
```

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
- **EMR Security Configuration must be created BEFORE the cluster — it cannot be applied to a running cluster.** If you need to add encryption to an existing unencrypted cluster, you must create a new cluster with the Security Configuration attached. There is no in-place upgrade path.
- **EC2 Instance Profile ≠ EMR Service Role:** Two different IAM roles with completely different purposes. Instance Profile = what your Spark/Hive code uses to call S3, Glue, DynamoDB. Service Role = what the EMR service uses to provision EC2 instances and VPC resources. Exam questions describe a symptom ("job can't read from S3 bucket") and ask which role to fix — almost always the **Instance Profile**.
- **HDFS replication factor and Core node count:** Default HDFS replication factor is **3**, meaning each block is stored on 3 different Core nodes. You need at least 3 Core nodes for full HDFS fault tolerance. With fewer than 3 Core nodes, HDFS under-replicates blocks. With 1 Core node, a single failure = data loss.
- **EMR Managed Scaling vs Auto Scaling:** Managed Scaling (newer, recommended) automatically adds and removes Task nodes based on YARN pending containers and memory metrics — zero configuration needed beyond min/max bounds. Auto Scaling (older) requires you to manually define CloudWatch alarms and scaling policies. **Exam pattern:** "automatically scale EMR cluster based on workload with least operational overhead" → Managed Scaling.
- **Cluster logging vs Step logging:** EMR cluster logs (YARN, HDFS, system daemons) go to S3 if you enable logging at cluster creation by specifying an S3 log URI. Step logs (your Spark job stdout/stderr) go to a subdirectory within that same S3 path. If you forget to set the log URI at cluster creation, there is **no way to add it later** — you must terminate and relaunch the cluster.

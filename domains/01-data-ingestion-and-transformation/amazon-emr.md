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

## Storage Options

| Option | Description |
|--------|-------------|
| **HDFS** | Local cluster storage, fast but ephemeral (lost when cluster terminates) |
| **EMRFS (S3)** | Persistent storage via S3. Recommended for data lakes. Supports consistent reads |
| **EBS** | Attached volumes for temporary storage |

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

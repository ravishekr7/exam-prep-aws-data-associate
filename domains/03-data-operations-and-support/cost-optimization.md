# Cost Optimization

Strategies for reducing AWS data engineering costs.

---

## Compute

| Strategy | Savings | Use Case |
|----------|---------|----------|
| **Spot Instances** | Up to 90% | EMR Task nodes, Glue Flex workers — stateless/fault-tolerant only |
| **Graviton (ARM)** | ~20% | RDS, Lambda, EMR, ECS, Redshift — same performance, lower price |
| **Reserved Instances (RI)** | Up to 72% | Steady-state workloads: RDS, Redshift, EMR Core nodes |
| **Savings Plans** | Up to 66% | Flexible commitment across EC2, Lambda, Fargate (Compute Savings Plan) |
| **Serverless** | Variable | Pay-per-use: Lambda, Athena, Glue, DynamoDB On-Demand, Step Functions |
| **Right-sizing** | Variable | Match instance type to actual CPU/memory usage |

### Spot Instances — Rules

- **Use for:** EMR Task nodes, batch jobs, stateless Lambda invocations, Fargate Spot
- **Never use for:** EMR Primary/Core nodes (data loss on interruption), RDS (not supported), stateful workloads
- Configure **interruption handling:** checkpoint to S3, use EMR Instance Fleets for diversification
- **Glue Flex workers:** Glue's Spot-like mode — up to 34% cheaper, may have startup delay

### Right-Sizing with AWS Compute Optimizer

- Analyzes CloudWatch utilization metrics (CPU, memory, network) over 14 days
- Recommends: downsize over-provisioned instances, upgrade under-provisioned ones
- Covers: EC2, Lambda, ECS Fargate, EBS volumes, Auto Scaling groups
- **Free to use** — no additional cost for recommendations
- Access via Console → Compute Optimizer or AWS CLI

### Reserved Instances vs Savings Plans

| Feature | Reserved Instances | Compute Savings Plans |
|---------|-------------------|----------------------|
| **Commitment** | Specific instance type/region | Spend amount ($/hour) |
| **Flexibility** | Low (locked to instance type/family) | High (any EC2, Lambda, Fargate) |
| **Discount** | Up to 72% | Up to 66% |
| **Best for** | Redshift, RDS (specific configs) | Mixed EC2 + Lambda workloads |
| **Capacity reservation** | Yes (regional or AZ) | No |

> **Exam tip:** Savings Plans are more flexible than RIs. Use RIs only when you need specific instance type/AZ capacity reservation (e.g., Redshift ra3.4xlarge in us-east-1).

---

## Storage (S3)

### Storage Classes & Lifecycle

| Class | Min Storage | Retrieval | Best For |
|-------|-------------|-----------|---------|
| **Standard** | None | Immediate | Frequently accessed data |
| **Standard-IA** | 30 days | Immediate | Infrequent access, rapid retrieval |
| **One Zone-IA** | 30 days | Immediate | Re-creatable data (lower cost, single AZ) |
| **Glacier Instant Retrieval** | 90 days | Milliseconds | Archives accessed once a quarter |
| **Glacier Flexible Retrieval** | 90 days | 1–12 hours | Archives with rare access |
| **Glacier Deep Archive** | 180 days | 12–48 hours | Compliance archives, rarely accessed |
| **Intelligent-Tiering** | None | Immediate | Unknown/changing access patterns |

### Lifecycle Policy Example

```json
{
  "Rules": [{
    "Transitions": [
      {"Days": 30,  "StorageClass": "STANDARD_IA"},
      {"Days": 90,  "StorageClass": "GLACIER"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 2555},
    "Status": "Enabled"
  }]
}
```

### Intelligent-Tiering Details

- Automatically moves objects between access tiers based on access patterns
- No retrieval fees (unlike IA/Glacier)
- Small monitoring fee per object (not cost-effective for objects < 128 KB)
- Add archive tiers: Deep Archive after 180 days of no access

### Data Format & Compression

| Format | Compression | Scan savings (Athena) | Storage savings |
|--------|------------|----------------------|----------------|
| CSV (uncompressed) | None | 0% | 0% |
| CSV (GZIP) | GZIP | 0% (no column skip) | ~70% |
| Parquet (Snappy) | Per column | ~85–95% | ~75% |
| ORC (Zlib) | Per column | ~85–95% | ~80% |

> Always use Parquet or ORC for Athena and Redshift Spectrum workloads — directly reduces query costs.

---

## Data Transfer

| Direction | Cost |
|-----------|------|
| **Inbound (Ingress)** | Free |
| **Outbound to Internet** | $0.09/GB (first 10 TB/month) |
| **Within same AZ** | Free (same-region, same-AZ EC2 traffic) |
| **Cross-AZ** | $0.01/GB each way |
| **Cross-region** | Varies by region pair |
| **To CloudFront** | Free |

### Reduce Data Transfer Costs

- **VPC Endpoints (Gateway):** Free — S3 and DynamoDB traffic stays on AWS backbone, avoids NAT Gateway ($0.045/GB)
- **VPC Endpoints (Interface):** Small hourly cost but saves on NAT Gateway for other services
- **PrivateLink vs VPC Endpoint:** PrivateLink is interface endpoint — use for services without gateway endpoints
- **S3 Transfer Acceleration:** Faster uploads via CloudFront edge — costs extra, use only when speed matters
- **Same-AZ placement:** Place EMR, Glue, Lambda, and S3 access in same region to avoid cross-region transfer costs

### NAT Gateway vs VPC Endpoint Savings

```
Without VPC endpoint:
  Lambda → NAT Gateway ($0.045/GB) → S3

With S3 VPC Gateway Endpoint:
  Lambda → S3 (free, private path)
```

For high-volume Lambda → S3 workflows, this can be significant savings.

---

## Service-Specific Cost Optimization

### Athena

| Technique | Why |
|-----------|-----|
| Parquet/ORC columnar format | Scan only queried columns — up to 95% cost reduction |
| Partition pruning | Skip entire partitions not matching the WHERE clause |
| Partition projection | Avoid Glue catalog scan overhead |
| Result reuse (caching) | Zero scan cost for repeated identical queries |
| Workgroup byte limits | Cap runaway ad-hoc queries |
| CTAS to Parquet | Convert expensive CSV sources once |

### Redshift

| Technique | Why |
|-----------|-----|
| RA3 node type | Managed storage — pay for compute and storage separately; scale independently |
| Concurrency Scaling | Add capacity for burst queries without permanently over-provisioning |
| Redshift Spectrum | Query S3 directly instead of loading cold data into Redshift |
| Pause/Resume (Serverless) | Redshift Serverless auto-pauses on inactivity |
| Compression (AZ64/ZSTD) | Reduce storage footprint and I/O |
| Materialized Views | Pre-compute expensive aggregations instead of re-scanning |

### DynamoDB

| Technique | Why |
|-----------|-----|
| On-Demand mode | Spiky or unpredictable traffic — no over-provisioning |
| Provisioned + Auto-Scaling | Predictable steady load — cheaper than On-Demand at scale |
| DynamoDB Standard-IA table class | Infrequently accessed tables — ~60% lower storage cost |
| Expire items (TTL) | Automatically delete old records — free, reduces storage |
| S3 export for analytics | Export to S3 + Athena instead of scanning DynamoDB directly |

### EMR

| Technique | Why |
|-----------|-----|
| Spot for Task nodes | Core nodes on Reserved/On-Demand, Task nodes on Spot |
| Transient clusters | Terminate cluster after job completes — no idle cost |
| EMR Managed Scaling | Right-sizes cluster automatically during job |
| Graviton instance types | ~20% better price-performance |
| Instance Fleets | Diversify Spot across instance types — reduces interruptions |
| S3 for HDFS | Use EMRFS + S3 instead of HDFS — persist data after cluster terminates |

### Glue

| Technique | Why |
|-----------|-----|
| Flex execution class | Up to 34% cheaper — uses Spot-like capacity |
| Auto-scaling DPUs | Pay for actual DPUs used, not fixed allocation |
| Job bookmarks | Avoid reprocessing already-processed data |
| G.1X workers (default) | Use G.2X only when jobs need more memory |
| Glue Data Catalog | Centralized metadata — avoid duplicate crawling costs |

### Lambda

| Technique | Why |
|-----------|-----|
| Right-size memory | Find optimal memory/duration combination (AWS Power Tuning) |
| Graviton2 (ARM) | Same performance, ~20% cheaper |
| Avoid over-provisioning concurrency | Reserved concurrency costs even when idle |
| Provisioned Concurrency | Only for latency-sensitive functions — not for batch |
| Batch processing | Larger batches (Kinesis, SQS) = fewer invocations = lower cost |

### Kinesis

| Technique | Why |
|-----------|-----|
| On-Demand → Provisioned (after stabilizing) | On-Demand is more expensive at sustained high throughput |
| Right-size shard count | Over-provisioned shards = wasted cost per shard-hour |
| Kinesis Firehose for delivery | No consumer application management; pay per GB delivered |
| Aggregated records | Batch small records in a single PutRecord call (KPL library) |

### Step Functions

| Technique | Why |
|-----------|-----|
| Express Workflows for high-volume | Per-execution pricing vs per-transition (much cheaper at scale) |
| Minimize state transitions | Combine steps where possible in Standard Workflows |
| Activity workers vs Lambda | Activity workers (polling) avoid Lambda invocation cost for long-running tasks |

---

## Cost Monitoring & Governance

### AWS Cost Explorer

- Visualize spending by service, account, region, tag
- **Rightsizing recommendations** built in for EC2
- **Savings Plans / RI recommendations** based on usage history
- Identify top cost drivers with time-series breakdown

### AWS Budgets

- Set budget alerts on **cost, usage, or coverage**
- Alert types: Actual vs Forecasted
- Budget actions: Restrict IAM policies, target EC2 Auto Scaling, notify SNS when threshold hit

### AWS Cost Anomaly Detection

- ML-based service that detects **unexpected spending spikes**
- Monitors by service, account, region, or cost category
- Sends alerts when anomaly detected (SNS or email)
- No threshold to configure — ML learns your baseline automatically
- Use case: catch a runaway Athena query scanning TBs, or a misconfigured Lambda loop

### Cost Allocation Tags

- Tag resources (e.g., `team=data-engineering`, `env=prod`, `project=pipeline-v2`)
- Activate tags in Billing Console — only then do they appear in Cost Explorer
- Use **AWS Organizations Cost Categories** to group accounts by business unit

### Unit Economics for Data Pipelines

Track cost-per-unit metrics to understand efficiency:

| Metric | How to Calculate | Why It Matters |
|--------|-----------------|---------------|
| Cost per GB processed | Total job cost / GB ingested | Benchmark pipeline efficiency |
| Cost per record | Total Athena + Glue cost / records output | Compare pipeline versions |
| Cost per dashboard query | Athena scan cost / queries | Justify SPICE vs Direct Query |
| Idle cluster cost | EMR/Redshift cluster hours × rate when no jobs run | Justify auto-termination |

---

## Cost Comparison: Athena vs Redshift vs EMR for Analytics

| Workload | Athena | Redshift | EMR |
|---------|--------|---------|-----|
| **Ad-hoc infrequent queries** | Best (pay per query) | Expensive (cluster running) | Expensive (cluster startup) |
| **High-concurrency dashboards** | Poor (each query scans) | Best (cache, WLM) | Not designed for this |
| **Daily batch ETL at scale** | OK for simple transforms | OK via COPY/UNLOAD | Best (Spark flexibility) |
| **Complex ML/custom processing** | Not suitable | Not suitable | Best |
| **Serverless / zero infra** | Best | OK (Serverless mode) | Poor |

---

## Exam Gotchas

- **"Most cost-effective"** = often involves S3 Intelligent-Tiering, Spot Instances, Athena pay-per-query, or serverless
- **Spot Instances = Task nodes only** (not EMR Primary or Core — data loss risk)
- **VPC Gateway Endpoints = free** and reduce NAT Gateway data processing charges
- **Inbound data transfer is always free** — only outbound costs money
- **Serverless ≠ always cheaper** — at high sustained throughput, provisioned is often cheaper
- **Savings Plans are more flexible** than RIs — prefer Savings Plans unless you need capacity reservation
- **AWS Cost Anomaly Detection** = ML-based spending spike alerts, no manual threshold needed
- **DynamoDB TTL = free** — always use to purge stale data and reduce storage cost
- **Glue Flex = Spot-like** — up to 34% cheaper but may have delayed start
- **Redshift RA3** separates compute and storage billing — don't pay for compute when not querying
- **Step Functions Express** is orders of magnitude cheaper at high invocation rates vs Standard
- **Compute Optimizer is free** — use it to identify over-provisioned EC2/Lambda before purchasing RIs

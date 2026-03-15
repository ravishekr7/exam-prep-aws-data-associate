# Cost Optimization

Strategies for reducing AWS data engineering costs.

---

## Compute

| Strategy | Savings | Use Case |
|----------|---------|----------|
| **Spot Instances** | Up to 90% | EMR Task nodes, Glue Flex workers. Stateless/fault-tolerant only |
| **Graviton Processors** | ~20% | RDS, Lambda, EMR, ECS. ARM-based, better price-performance |
| **Reserved Instances** | Up to 72% | Predictable, steady-state workloads (RDS, Redshift, EMR Core) |
| **Savings Plans** | Up to 72% | Flexible commitment across compute services |
| **Serverless** | Variable | Pay-per-use: Lambda, Athena, Glue, DynamoDB On-Demand |

---

## Storage (S3)

- **Lifecycle Policies:** Automate transitions: Standard -> IA (30d) -> Glacier (90d) -> Expire
- **Intelligent-Tiering:** When access patterns are unknown (no retrieval fees)
- **Compression:** Snappy/GZIP reduces storage size
- **Columnar Formats:** Parquet/ORC reduce query scan size (Athena billing)

---

## Data Transfer

| Direction | Cost |
|-----------|------|
| **Inbound (Ingress)** | Free |
| **Outbound (Egress)** | Paid |
| **Within same AZ** | Usually free |
| **Cross-AZ** | Paid |
| **VPC Endpoints** | Avoids NAT Gateway charges, keeps traffic on AWS backbone |

---

## Service-Specific Tips

| Service | Optimization |
|---------|-------------|
| **Athena** | Parquet + Partitioning + Compression = reduce TB scanned |
| **Redshift** | Use RA3 (managed storage), Concurrency Scaling for bursts |
| **DynamoDB** | Auto-Scaling or On-Demand. Scheduled Scaling for predictable spikes |
| **EMR** | Spot for Task nodes, Graviton instances, transient clusters |
| **Glue** | Flex workers (Spot-like), Auto-scaling DPUs |
| **Lambda** | Right-size memory, use Graviton (ARM), avoid over-provisioning concurrency |
| **Kinesis** | On-Demand mode vs over-provisioned shards |

---

## Monitoring Cost

- **AWS Cost Explorer:** Analyze spending patterns
- **AWS Budgets:** Set alerts when spending exceeds thresholds
- **Cost Allocation Tags:** Tag resources for cost attribution

---

## Exam Gotchas

- **"Most cost-effective"** answers often involve: S3 Intelligent-Tiering, Spot Instances, Athena (pay-per-query)
- Spot Instances ONLY for fault-tolerant workloads (EMR Task nodes, not Primary)
- VPC Endpoints reduce NAT Gateway data processing charges
- Inbound data transfer is always free
- Serverless doesn't always mean cheaper - depends on usage pattern

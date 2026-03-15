# Amazon S3 - Data Lake Storage

S3 is the central repository for all data (structured, semi-structured, unstructured) in an AWS Data Lake.

---

## Storage Classes

| Class | Access | Retrieval | Min Duration | Use Case |
|-------|--------|-----------|-------------|----------|
| **Standard** | Frequent | ms | None | Hot data, frequent access |
| **Intelligent-Tiering** | Auto-managed | ms | None | Unknown access patterns (exam favorite) |
| **Standard-IA** | Infrequent | ms | 30 days | Known infrequent access |
| **One Zone-IA** | Infrequent, 1 AZ | ms | 30 days | Re-creatable data, 20% cheaper than IA |
| **Glacier Instant Retrieval** | Archive | ms | 90 days | Archive but need instant access |
| **Glacier Flexible Retrieval** | Archive | Minutes-hours | 90 days | Backups, compliance data |
| **Glacier Deep Archive** | Rare archive | 12-48 hours | 180 days | Regulatory archives, lowest cost |

### Intelligent-Tiering (Exam Favorite)
- Moves data between frequent/infrequent access tiers automatically
- No retrieval fees
- Small monitoring fee per object
- Best default choice when access patterns are unknown

---

## Data Lake Design

### Partitioning
Organize data in folders: `s3://bucket/table/year=2023/month=01/day=21/`

- Drastically reduces data scanned by Athena/Redshift Spectrum/Glue (**Partition Pruning**)
- Results in cheaper and faster queries
- Choose partition keys based on query patterns (usually date-based)

### Partition Projection (Athena)
- Define partition patterns in table properties instead of crawling
- Athena calculates partitions at query time
- Eliminates need for Glue Crawlers to discover partitions

---

## S3 Lifecycle Policies

Automate storage class transitions:
```
Standard -> Standard-IA (30 days) -> Glacier Flexible (90 days) -> Expire (365 days)
```

- Transition rules: Move objects between classes after N days
- Expiration rules: Delete objects after N days
- Can filter by prefix or tags
- Minimum 30 days before transitioning from Standard to IA classes

---

## S3 Versioning

- Keeps all versions of an object
- Protects against accidental deletion (delete marker instead of permanent delete)
- Combined with MFA Delete for extra protection
- Required for Cross-Region Replication (CRR)

---

## S3 Cross-Region Replication (CRR)

- Replicate objects to another region for DR or latency
- Requires versioning enabled on both buckets
- Can replicate to different storage class
- Use for disaster recovery and compliance (data sovereignty)

---

## Exam Gotchas

- **Intelligent-Tiering** = answer when "access patterns are unknown" or "minimize retrieval cost risk"
- **Lifecycle policies** = "automate cost optimization" or "move data to cheaper storage over time"
- Glacier Deep Archive = cheapest but 12-48 hour retrieval (not for any real-time needs)
- One Zone-IA = only for re-creatable data (less durable)
- S3 requests are per-prefix: 3,500 PUT and 5,500 GET per second per prefix
- **S3 Object Lock** for WORM (Write Once Read Many) compliance requirements

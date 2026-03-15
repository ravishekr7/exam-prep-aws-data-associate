# Amazon MSK (Managed Streaming for Apache Kafka)

Fully managed Apache Kafka service on AWS.

---

## When to Use MSK vs Kinesis

| Feature | Amazon MSK | Kinesis Data Streams |
|---------|-----------|---------------------|
| **Protocol** | Native Kafka APIs | AWS SDK / KPL / KCL |
| **Migration** | Existing Kafka workloads | New AWS-native projects |
| **Configuration** | Full Kafka tuning | Limited |
| **Storage** | Configurable (tiered storage) | Up to 365 days |
| **Consumer Groups** | Kafka consumer groups | KCL / Lambda |
| **Management** | Semi-managed (broker/ZooKeeper) | Fully managed |

### Use MSK When
- Migrating existing Kafka workloads to AWS
- Need Kafka-specific features (consumer groups, compaction, custom configs)
- Team has Kafka expertise

### Use Kinesis When
- Building new AWS-native streaming applications
- Want minimal management overhead
- Need tight integration with Lambda, Firehose, Flink

---

## MSK Modes

- **MSK Provisioned:** You choose broker types and count
- **MSK Serverless:** Auto-scales, no broker management, pay per throughput
- **MSK Connect:** Run Kafka Connect connectors as managed service (source/sink connectors)

---

## Key Concepts

- **Topics:** Logical channel for messages
- **Partitions:** Parallelism unit within a topic (like Kinesis shards)
- **Consumer Groups:** Multiple consumers share work on a topic
- **Log Compaction:** Keep only the latest value per key (deduplication)

---

## Exam Gotchas

- MSK is the answer when the question mentions "existing Kafka" or "Kafka migration"
- MSK Connect is for managed Kafka connectors (no need to run your own)
- MSK Serverless removes broker management entirely
- MSK supports IAM authentication, TLS, and SASL/SCRAM

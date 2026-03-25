# Amazon Kinesis

Real-time data streaming services.

---

## 1. Kinesis Data Streams (KDS)

Massively scalable, durable real-time data streaming service.

### Architecture
- **Shards:** Unit of throughput
  - **Write:** 1 MB/sec or 1,000 records/sec per shard
  - **Read:** 2 MB/sec per shard (Shared) or 2 MB/sec per consumer (Enhanced Fan-out)
- **Record:** Sequence Number + Partition Key (determines shard) + Data Blob (base64)

### Capacity Modes
| Mode | Description | Best For |
|------|-------------|----------|
| **Provisioned** | You define shard count. Pay per shard/hour | Predictable traffic |
| **On-Demand** | AWS auto-scales. Higher base cost | Unpredictable/spikey traffic |

### Retention
- Default: 24 hours
- Maximum: 365 days
- Extended retention costs extra

### Producers and Consumers
**Producers:**
- KPL (Kinesis Producer Library) - efficient batching
- SDK - low level control
- Kinesis Agent - standalone Java app, monitors log files

**Consumers:**
- KCL (Kinesis Client Library) - handles checkpointing via DynamoDB
- Lambda
- Firehose
- Flink (Kinesis Data Analytics)

---

## 2. Kinesis Shard Math

> **This section is heavily tested. Know the formula and be able to work examples in your head.**

### Shard Sizing Formula

```
shards_needed = max(
    ceil(ingest_MB_per_sec / 1),          # Write: 1 MB/s per shard
    ceil(read_MB_per_sec / 2),             # Read: 2 MB/s per shard (shared)
    ceil(records_per_sec / 1000)           # Write: 1,000 records/s per shard
)
```

Always use the **maximum** of the three constraints — whichever is the binding limit.

### Worked Example
**Scenario:** 3.5 MB/s ingest rate. 4 downstream consumers, each reading 1 MB/s.

```
Write constraint:  ceil(3.5 / 1)  = 4 shards
Read constraint:   ceil(4 / 2)    = 2 shards   (total read = 4 × 1 = 4 MB/s)
Records constraint: assume 500 records/s → ceil(500/1000) = 1 shard

Answer: max(4, 2, 1) = 4 shards
```

> Note: If those 4 consumers use **Enhanced Fan-Out**, each gets their own 2 MB/s per shard, so the read constraint changes to `ceil(1/2) = 1 shard` per consumer. The binding constraint becomes the write side: **4 shards**.

### Shard Split vs Shard Merge
| Operation | Effect | When to Use |
|-----------|--------|-------------|
| **Split** | One shard → Two shards. Doubles capacity for that shard. | Traffic growing, hitting write limits (`ProvisionedThroughputExceeded`) |
| **Merge** | Two shards → One shard. Halves capacity and cost. | Traffic decreased, over-provisioned, reducing cost |

> Splits and merges are **not instant** — the original shard goes into CLOSED state and eventually expires. New shards become ACTIVE. Allow up to 30 seconds.

### Hot Shard Problem
**Symptom:** One shard receives a disproportionate share of traffic because many records share the same partition key (e.g., all records from one device, or all events for one user ID).

**Solution: Add a random suffix to the partition key**
```python
import random

# Bad: all records for user 123 go to the same shard
partition_key = "user_123"

# Good: spread user 123 across up to 10 shards
partition_key = f"user_123-{random.randint(0, 9)}"
```

**Tradeoff:** Records for the same logical entity are no longer on the same shard, so strict ordering across the entity is lost. Acceptable for most analytics workloads.

---

## 3. Enhanced Fan-Out

Enhanced Fan-Out (EFO) is a Kinesis feature that gives **each registered consumer its own dedicated read throughput** per shard, instead of sharing the shard's read bandwidth.

### Standard vs Enhanced Fan-Out Comparison

| Aspect | Standard GetRecords | Enhanced Fan-Out |
|--------|--------------------|--------------------|
| **Throughput** | 2 MB/s **shared** across ALL consumers per shard | 2 MB/s **dedicated** per registered consumer per shard |
| **Latency** | ~200ms (polling) | ~70ms (push-based) |
| **API** | `GetRecords` (polling, max 5 calls/sec per shard) | `SubscribeToShard` (HTTP/2 push) |
| **Scaling limit** | With 5 consumers: each gets max 0.4 MB/s | With 5 consumers: each still gets 2 MB/s |
| **Cost** | No extra charge beyond shard cost | Additional charge per shard-hour per consumer |

### When to Use Enhanced Fan-Out
- You have **multiple consumers** (e.g., Lambda for real-time alerts + Flink for aggregations + Firehose for archiving) that all need full throughput.
- With standard mode and 5 consumers, each only gets 400 KB/s per shard. EFO restores the full 2 MB/s per consumer.
- **Exam pattern:** "5 downstream consumers each need to read at full speed from the same stream" → Enable Enhanced Fan-Out.

---

## 4. Managed Service for Apache Flink

Previously called **Kinesis Data Analytics**. Fully managed Apache Flink for real-time stream processing. You write applications in Java, Python, or Scala (or use Flink SQL), and AWS handles the infrastructure, scaling, and checkpointing.

### What It Is
- Managed Flink cluster: no servers to provision, auto-scales based on parallelism settings
- Inputs: Kinesis Data Streams, Amazon MSK (Kafka)
- Outputs: Kinesis Data Streams, Firehose, S3, DynamoDB, OpenSearch, custom sinks
- Sub-second processing latency

### Flink vs Lambda: When to Use Each

| Aspect | Managed Service for Apache Flink | AWS Lambda |
|--------|----------------------------------|------------|
| **State** | Stateful — Flink maintains state across records across time | Stateless — each invocation is independent |
| **Windowing** | Native tumbling, sliding, session windows | Must implement manually (complex) |
| **Pattern Matching (CEP)** | Built-in Complex Event Processing library | Must build custom logic |
| **Throughput** | Very high (parallel, distributed) | Limited by concurrency and 15-min timeout |
| **Use case** | Fraud detection, clickstream analytics, IoT aggregations, anomaly detection | Simple per-record transformations, lightweight enrichment |

### Key Concept: Stateful Processing
Flink maintains **state** — it can remember what happened in previous records and over time windows. Lambda cannot do this natively.

```
Example: "Flag a user who fails login more than 5 times in a 10-minute window"
→ Flink: maintain a count per user_id in a 10-minute sliding window. Trivial.
→ Lambda: must store counts externally (DynamoDB), coordinate across invocations. Complex.
```

### Exam Pattern
> "Real-time fraud detection with pattern matching across a time window" → **Managed Service for Apache Flink**, not Lambda.
> "Stateful stream processing", "windowed aggregations", "CEP" → Flink.

---

## 5. Kinesis Data Firehose (Delivery Stream)

Fully managed service to **load** streaming data into destinations. **Near real-time** (minimum 60s latency or buffer size).

### Destinations (Important for Exam)
1. Amazon S3
2. Amazon Redshift (via intermediate S3 bucket)
3. Amazon OpenSearch Service
4. Splunk
5. HTTP Endpoint (Datadog, MongoDB, 3rd party)

### Capabilities
- **Transformation:** Invoke Lambda to transform records before delivery (e.g., JSON to CSV, masking PII)
- **Format Conversion:** JSON to Parquet/ORC (requires Glue Table definition) - essential for S3/Athena optimization
- **Buffering:** Configure by time (60-900s) or size (1-128 MB)

### Cost
Pay for data ingested. No idle cost. No shards to manage.

---

## 6. Kinesis vs MSK vs SQS — Full Comparison

| Feature | Kinesis Data Streams | Amazon MSK (Kafka) | Amazon SQS |
|---------|---------------------|--------------------|------------|
| **Ordering** | Ordered within a shard | Ordered within a partition | No ordering (FIFO queue has ordering) |
| **Consumer model** | Multiple consumers share stream; EFO for dedicated throughput | Consumer groups — each group gets all messages; within a group, one consumer per partition | Single consumer per message (message deleted after consumption) |
| **Replay capability** | Yes — re-read from any position within retention window (up to 365 days) | Yes — configurable retention, re-read from any offset | No — once consumed and deleted, gone |
| **Max retention** | 365 days | Unlimited (storage-based) | 14 days |
| **Throughput model** | Shard-based (1 MB/s write, 2 MB/s read per shard) | Partition-based, highly scalable | Virtually unlimited (auto-scales) |
| **Latency** | ~200ms (standard) / ~70ms (EFO) | Configurable (sub-100ms possible) | Best-effort (milliseconds to seconds) |
| **Migrate from Kafka** | Not a drop-in replacement | **Yes — use MSK** (same protocol) | No |
| **When to choose** | AWS-native streaming, IoT, log aggregation, real-time analytics | Existing Kafka ecosystem, on-prem Kafka migration, Kafka Streams, MirrorMaker | Job queues, task distribution, decoupling microservices, fan-out with SNS |

---

## KDS vs Firehose Comparison

| Feature | Kinesis Data Streams | Kinesis Firehose |
|---------|---------------------|------------------|
| **Type** | Custom streaming code | ETL / Storage loader |
| **Latency** | Real-time (~200ms) | Near real-time (60s min) |
| **Storage** | Durable (up to 365 days) | No storage (delivery only) |
| **Management** | Manual (Shards) / On-Demand | Fully managed |
| **Destination** | Custom consumers, Spark, Flink | S3, Redshift, OpenSearch, Splunk |
| **Transformation** | Consumer handles it | Built-in Lambda transform |
| **Scaling** | Shard splitting/merging | Automatic |

---

## Exam Gotchas

- **Kinesis Agent vs KPL:** Agent is easiest for server logs (install and go). KPL is for building custom high-performance producers
- **"Near real-time to S3"** = Firehose (always)
- **"Sub-second stream processing"** = Flink (Kinesis Data Analytics)
- **ProvisionedThroughputExceeded:** Fix by resharding (split shards) or switching to On-Demand
- **Iterator Age (CloudWatch metric):** Shows how far behind a consumer is. High age = lagged processing. Fix: increase shards, optimize consumer, increase Lambda batch size
- Firehose does NOT support Kinesis Data Streams as a destination (it's the other way around)
- Firehose writes to Redshift via an intermediate S3 bucket (COPY command)
- **Kinesis Firehose minimum latency is 60 seconds** (minimum buffer interval). If a requirement says "deliver within 30 seconds" or "sub-minute delivery", Firehose is **NOT** the right answer — use KDS with a Lambda consumer writing directly to the destination.
- **Firehose can transform records with Lambda** before delivery — this is a very common exam pattern. Lambda receives a batch from Firehose, transforms the records (e.g., format conversion, PII masking), and returns them. Transformed records are then written to the destination.
- **KCL (Kinesis Client Library)** automatically handles checkpointing (tracks last processed sequence number in DynamoDB), load balancing across multiple consumer instances, and shard leasing. On the exam, KCL is preferred over raw Lambda consumers when you need sophisticated consumer management.
- **IteratorAgeMilliseconds** CloudWatch metric: measures how far behind the tip of the stream the consumer is. If this metric is **increasing over time**, your consumer is falling behind the stream — scale your consumers (more Lambda concurrency, more KCL workers) or increase the number of shards.

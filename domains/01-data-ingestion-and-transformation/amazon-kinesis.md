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

### Enhanced Fan-Out
- Dedicated 2 MB/sec throughput **per consumer** (not shared)
- Uses HTTP/2 push instead of polling
- Use when you have multiple consumers that need low latency

---

## 2. Kinesis Data Firehose (Delivery Stream)

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

## 3. Kinesis Data Analytics (Managed Apache Flink)

- **Use for:** Stateful processing, sliding windows, complex analytics inside the stream
- **Input:** Kinesis Streams, Firehose
- **Output:** Kinesis Streams, Firehose, S3, DynamoDB
- Sub-second processing latency

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

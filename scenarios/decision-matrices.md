# Decision Matrices

"When to use What" logic for exam scenario questions.

---

## Streaming: Kinesis Data Streams vs Firehose vs MSK vs SQS

| Feature | Kinesis Data Streams | Kinesis Firehose | Amazon MSK (Kafka) | Amazon SQS |
|---------|---------------------|------------------|--------------------|------------|
| **Ordering** | Ordered within a shard | No ordering guarantee | Ordered within a partition | No ordering (FIFO queue option available) |
| **Consumer model** | Multiple consumers can read simultaneously; EFO for dedicated per-consumer throughput | Delivers to one destination (S3, Redshift, OpenSearch, Splunk) | Consumer groups — each group gets all messages independently | Single consumer per message (deleted after consume) |
| **Replay capability** | Yes — re-read any position within retention window | No — delivery only, no storage | Yes — re-read from any offset within retention | No — consumed messages are gone |
| **Max retention** | Up to 365 days | No retention (buffers for seconds to minutes then delivers) | Unlimited (storage-based pricing) | 14 days |
| **Throughput model** | Shard-based: 1 MB/s write, 2 MB/s read per shard | Fully managed, no throughput limits to manage | Partition-based, highly configurable | Virtually unlimited (auto-scales) |
| **Minimum latency** | ~70ms (EFO) / ~200ms (standard) | **60 seconds minimum** (buffer interval) | Sub-100ms possible | Milliseconds to seconds |
| **"Migrate from existing Kafka"** | No — different protocol | No | **Yes — MSK is Kafka-compatible** | No |
| **When to choose** | AWS-native real-time streaming; IoT ingestion; log analytics; replay needed; multiple consumers | Near-real-time delivery to S3, Redshift, or OpenSearch; format conversion; Lambda transform before delivery | Existing Kafka apps; MirrorMaker replication; Kafka Streams; custom partition strategies | Job queues; task distribution; microservice decoupling; fan-out via SNS |

---

## Analytics Compute: Glue vs EMR vs Lambda vs Athena

| Feature | AWS Glue | Amazon EMR | AWS Lambda | Amazon Athena |
|---------|----------|------------|------------|---------------|
| **Workload type** | Batch ETL, schema mapping, data catalog | Batch + streaming; any Hadoop ecosystem job | Event-driven, lightweight, per-record transforms | Ad-hoc SQL queries on S3 data |
| **Latency** | Cold start 10-30s; batch runs minutes | Cluster start minutes; jobs then execute | Milliseconds (warm) / seconds (cold) | Query-dependent; seconds to minutes |
| **Cost model** | Per DPU-second | Per EC2 instance-hour (Spot available) | Per request + GB-second of compute | Per TB of data scanned |
| **Management overhead** | Serverless — zero infrastructure | High (cluster management) or low (Serverless) | Serverless — zero infrastructure | Serverless — zero infrastructure |
| **Spark support** | Yes (PySpark, Scala, with DynamicFrames) | Yes (full control, custom JARs, all Spark configs) | No (but can call Spark via Step Functions) | No (SQL engine, not Spark) |
| **SQL support** | Limited (can run SQL via Spark) | Yes (Hive SQL, Spark SQL, Presto/Trino) | No (must write custom code) | **Yes — primary interface** |
| **Best for** | Serverless ETL pipelines, schema transformation, Glue catalog integration | Heavy batch workloads requiring custom frameworks, Spot Instances for cost, Hive/Presto/HBase | Lightweight per-event processing; triggers from S3, SQS, Kinesis; simple transformations | Infrequent or exploratory SQL queries on data in S3; cost-effective analytics without a warehouse |

---

## Streaming: Kinesis vs MSK vs SQS

| Feature | Kinesis Data Streams | Amazon MSK (Kafka) | SQS |
|---------|---------------------|-------------------|-----|
| **Nature** | Real-time stream | Real-time stream | Message queue (decoupling) |
| **Ordering** | Ordered within shard | Ordered within partition | No (unless FIFO) |
| **Consumers** | Multiple (fan-out) | Multiple (consumer groups) | Single (message deleted after consume) |
| **Retention** | Up to 365 days | Configurable (storage-based) | Up to 14 days |
| **Use When** | AWS-native streaming, IoT | Existing Kafka migration, custom config | Job queues, decoupling microservices |

---

## Analytics Storage: Redshift vs Athena vs EMR Spark vs OpenSearch

| Feature | Amazon Redshift | Amazon Athena | Amazon EMR Spark | Amazon OpenSearch |
|---------|----------------|---------------|------------------|-------------------|
| **Data location** | Redshift Managed Storage (RA3) or S3 (Spectrum) | **S3 only** | HDFS or S3 | OpenSearch domain (managed) |
| **Query type** | Complex SQL: JOINs, window functions, aggregations | SQL on structured/semi-structured data in S3 | SQL (Spark SQL), ML, custom code | Full-text search, log analytics, aggregations |
| **Latency** | Low (optimized for repeated complex queries) | Medium (scan-based, improves with partitioning) | Variable (depends on cluster size and job complexity) | Low for search (sub-second) |
| **Cost model** | Per node/hour (provisioned) or RPU/second (serverless) | Per TB of data scanned | Per instance-hour | Per instance-hour + storage |
| **Scales to** | Petabytes (RA3) | Exabytes (S3 is unlimited) | Petabytes (with enough nodes) | Terabytes (typical) |
| **Concurrency** | High (WLM queues, Concurrency Scaling) | Limited (quota on concurrent queries) | Depends on cluster sizing | High for search |
| **Best for** | BI dashboards, complex analytical queries, data warehouse workloads | Occasional SQL queries on S3; no loading required | Heavy transformations, ML feature engineering, complex ETL with Spark | Application search, log/metrics analysis, observability |

---

## Analytics Compute: Lambda vs Glue vs EMR

| Feature | Lambda | Glue | EMR |
|---------|--------|------|-----|
| **Time Limit** | 15 minutes | 48 hours | Unlimited |
| **Startup** | ms (warm) / sec (cold) | 10-30 seconds | Minutes |
| **Cost** | Per request/GB-sec | Per DPU-hour | Per instance-hour |
| **Use When** | Event triggers, lightweight logic | Serverless ETL, schema mapping | Heavy long-running batch, custom JARs |

---

## Database: DynamoDB vs RDS/Aurora vs DocumentDB vs Redshift vs Neptune vs MemoryDB

| Need | DynamoDB | RDS / Aurora | DocumentDB | Redshift | Neptune | MemoryDB |
|------|----------|-------------|------------|----------|---------|----------|
| **Data model** | Key-value, document | Relational (tables, rows) | Document (JSON/MongoDB) | Relational (columnar) | Graph (nodes, edges) | Key-value (Redis) |
| **Consistency** | Eventual (default) or strong | Strong (ACID) | Eventual | Strong (within cluster) | Strong | Strong |
| **Query pattern** | Single-item lookups by key; simple queries | Complex SQL JOINs, transactions | MongoDB queries, flexible schema | Complex analytical SQL, aggregations | Graph traversals (Gremlin, SPARQL, openCypher) | Simple key lookups, sorted sets, pub/sub |
| **Latency** | Single-digit milliseconds | Milliseconds to seconds | Milliseconds | Seconds to minutes | Sub-second | **Microseconds** |
| **Analytics support** | Limited (export to S3 or DynamoDB Streams) | Limited (read replicas for analytics) | Limited | **Primary use case** | Limited | None |
| **Best for** | High-throughput OLTP at any scale, serverless apps, session data | Traditional applications needing ACID; MySQL/PostgreSQL compatibility | MongoDB migration; flexible document storage | BI reporting, data warehouse, complex joins | Social networks, knowledge graphs, recommendation engines, fraud detection with graph relationships | Ultra-low-latency caching; session store; leaderboards; pub/sub |

---

## ETL Approach: Batch vs Streaming

| Factor | Batch | Streaming |
|--------|-------|-----------|
| **Latency Tolerance** | Hours acceptable | Seconds/minutes needed |
| **Volume** | Large periodic loads | Continuous small records |
| **Tools** | Glue, EMR, DMS Full Load | Kinesis, MSK, Flink, Firehose |
| **Cost** | Usually cheaper per record | Higher but enables real-time |

---

## Orchestration: Step Functions (Standard) vs Step Functions (Express) vs MWAA vs EventBridge Pipes

| Feature | Step Functions Standard | Step Functions Express | Amazon MWAA | EventBridge Pipes |
|---------|------------------------|----------------------|-------------|------------------|
| **Max duration** | Up to **1 year** | Up to **5 minutes** | Days to months (always-running environment) | Seconds to minutes |
| **Execution semantics** | **Exactly-once** (Standard workflows guarantee each state is executed once) | **At-least-once** (may execute more than once — must design idempotently) | Airflow semantics (task retries, depends_on_past) | At-least-once |
| **Audit history** | Full execution history in console + CloudWatch (up to 90 days) | Only CloudWatch logs (no step-level history in console for high-volume) | Airflow web UI with full task history, Gantt charts | CloudWatch only |
| **Cost model** | Per state transition (expensive for high-frequency workflows) | Per execution + duration (cheap for high-frequency) | Per environment-hour (always on, idle cost) | Per event processed |
| **Use case** | Long-running multi-step workflows; human approval steps; distributed transactions | High-volume event processing (thousands/sec); IoT, streaming pipelines | Complex DAG-based scheduling with backfill; existing Airflow teams | Point-to-point event routing between AWS services; simple filtering + enrichment |
| **When to choose** | Multi-service orchestration needing exact-once guarantees; audit trail required; complex branching | Microservices choreography needing fast orchestration at high volume | Team has Airflow expertise; need backfill; hundreds of task dependencies; Python operators | Replace custom EventBridge → Lambda → SQS plumbing with a managed pipe |

---

## Encryption: SSE-S3 vs SSE-KMS vs SSE-C vs Client-Side Encryption

| Feature | SSE-S3 (AES-256) | SSE-KMS | SSE-C | Client-Side Encryption |
|---------|-----------------|---------|-------|------------------------|
| **Key management** | AWS fully manages keys (no visibility into key) | **You manage CMK in KMS** (create, rotate, apply policies) | **You provide encryption key** with every request — AWS uses it, never stores it | You manage keys entirely; encryption/decryption happens before data reaches AWS |
| **Audit trail** | No key-level audit | **Yes — every use of the CMK logged in CloudTrail** | No AWS-side audit (key never stored) | Depends on your key management system |
| **Performance** | Fastest (no KMS API call) | Slight overhead (KMS API call per operation; throttling possible) | AWS performs encryption, but you provide key per request | Most overhead (encryption on client before upload) |
| **Cost** | Free (included in S3) | KMS API charges (~$0.03 per 10,000 requests) + CMK cost ($1/month per key) | Free (you provide the key) | Free from AWS perspective |
| **Compliance use case** | Basic encryption at rest; no audit requirement | **HIPAA, PCI-DSS, SOC** — need proof of key control + audit trail | When you need to control the key and don't want it stored in AWS | Maximum control (key never leaves your environment) |
| **When to choose** | Default for non-sensitive data | When you need key rotation, access policies, audit logs — most enterprise use cases | Specialized compliance where key cannot be stored in any AWS system | Highly regulated industries where data must be encrypted before reaching S3 |

---

## Database: When to Use What

| Need | Service |
|------|---------|
| OLTP + relational | RDS / Aurora |
| Key-value, millisecond latency | DynamoDB |
| Microsecond latency (cached) | DAX / MemoryDB |
| SQL analytics, complex joins | Redshift |
| Ad-hoc SQL on S3 | Athena |
| Graph relationships | Neptune |
| MongoDB compatibility | DocumentDB |
| Cassandra compatibility | Keyspaces |
| Full-text search, log analytics | OpenSearch |

---

## Orchestration: Step Functions vs MWAA vs Glue Workflows

| Need | Service |
|------|---------|
| Serverless multi-service workflow | Step Functions |
| Complex dependencies + backfilling | MWAA (Airflow) |
| Glue-only jobs and crawlers | Glue Workflows |
| Event routing and scheduling | EventBridge |

---

## File Format Decision

| Need | Format |
|------|--------|
| Analytics queries (Athena, Spectrum) | Parquet |
| Schema evolution in streaming | Avro |
| Hive on EMR | ORC |
| Simple data exchange | CSV |
| Semi-structured nested data | JSON |

---

## Exam Keywords -> Service Mapping

| Keyword | Service |
|---------|---------|
| "Serverless SQL" | Athena |
| "Complex joins, BI dashboards" | Redshift |
| "Sub-second latency" | DynamoDB / DAX |
| "Near real-time to S3" | Kinesis Firehose |
| "Real-time stream processing" | Kinesis Data Analytics (Flink) |
| "Existing Kafka" | MSK |
| "Schema evolution" | Glue Schema Registry / Avro |
| "Column/row level security" | Lake Formation |
| "Visualization/dashboards" | QuickSight |
| "SaaS data source" | AppFlow |
| "Migrate database" | DMS |
| "Global low-latency" | DynamoDB Global Tables / Aurora Global |
| "PII in S3" | Macie |
| "Track API calls" | CloudTrail |
| "Track config changes" | AWS Config |
| "Audit encryption key usage" | SSE-KMS + CloudTrail |
| "Least operational overhead + Iceberg" | S3 Tables |
| "Stateful stream processing" | Managed Service for Apache Flink |
| "At-least-once high-volume orchestration" | Step Functions Express |
| "Exactly-once long-running orchestration" | Step Functions Standard |
| "Existing Airflow / backfill" | MWAA |

# Decision Matrices

"When to use What" logic for exam scenario questions.

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

## Analytics Storage: Redshift vs Athena vs EMR

| Feature | Redshift | Athena | EMR |
|---------|----------|--------|-----|
| **Type** | Data warehouse | Serverless SQL | Big data platform |
| **Storage** | Managed (RA3) + S3 | S3 only | HDFS + S3 |
| **Performance** | High (indexed, cached) | Variable (scan-based) | High (tunable) |
| **Cost Model** | Per node/hour | Per TB scanned | Per instance/hour |
| **Use When** | BI reporting, complex joins, dashboards | Ad-hoc queries, infrequent analysis | Heavy transforms, ML, existing Hadoop |

---

## Compute: Lambda vs Glue vs EMR

| Feature | Lambda | Glue | EMR |
|---------|--------|------|-----|
| **Time Limit** | 15 minutes | 48 hours | Unlimited |
| **Startup** | ms (warm) / sec (cold) | 10-30 seconds | Minutes |
| **Cost** | Per request/GB-sec | Per DPU-hour | Per instance-hour |
| **Use When** | Event triggers, lightweight logic | Serverless ETL, schema mapping | Heavy long-running batch, custom JARs |

---

## ETL Approach: Batch vs Streaming

| Factor | Batch | Streaming |
|--------|-------|-----------|
| **Latency Tolerance** | Hours acceptable | Seconds/minutes needed |
| **Volume** | Large periodic loads | Continuous small records |
| **Tools** | Glue, EMR, DMS Full Load | Kinesis, MSK, Flink, Firehose |
| **Cost** | Usually cheaper per record | Higher but enables real-time |

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

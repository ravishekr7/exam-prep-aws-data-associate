# Architecture Patterns

Common real-world architectures tested in the exam.

---

## Pattern A: Modern Data Lakehouse (Batch)

```
Sources                    Ingest                Process              Consume
--------                   ------                -------              -------
RDS (MySQL)  --> DMS (CDC) --> S3 (Raw)  --> Glue ETL --> S3 (Clean/Parquet) --> Redshift Spectrum
Logs (files) --> Firehose  --> S3 (Raw)  --> Glue ETL --> S3 (Clean/Parquet) --> Athena
                                                                              --> QuickSight
Orchestration: Step Functions
Catalog: Glue Data Catalog
Security: Lake Formation
```

### Key Design Decisions
- DMS with CDC for ongoing replication without impacting source
- Firehose converts JSON to Parquet during delivery
- Step Functions orchestrates: Crawler -> ETL -> Validation -> Notification
- Lake Formation enforces column/row-level access

---

## Pattern B: Real-Time Analytics

```
Sources                  Ingest              Process                  Store
--------                 ------              -------                  -----
IoT Sensors  --> KDS --> Flink (windowing) --> DynamoDB (hot dashboard)
Clickstream  --> KDS --> Flink (analytics) --> Firehose --> S3 (cold archive)
```

### Key Design Decisions
- KDS for true real-time ingestion
- Flink for stateful processing (sliding windows, aggregations)
- Hot path: DynamoDB for live dashboards (sub-second reads)
- Cold path: Firehose to S3 for historical analysis

---

## Pattern C: CDC Pipeline (On-Prem to Cloud)

```
On-Prem Oracle --> SCT (schema convert) --> Aurora PostgreSQL
On-Prem Oracle --> DMS (Full Load + CDC) --> Aurora PostgreSQL
                                         --> S3 (for analytics via DMS)
```

### Key Design Decisions
- SCT first to convert schema for heterogeneous migration
- DMS handles ongoing replication
- Parallel CDC to S3 enables analytics without touching production DB

---

## Pattern D: Log Analytics Platform

```
Application Logs --> Firehose (5 min buffer, Parquet) --> S3 --> Athena (ad-hoc)
                                                              --> QuickSight (dashboard)
VPC Flow Logs --> S3 --> Athena
CloudTrail    --> S3 --> Athena
```

### Key Design Decisions
- Firehose buffering (5 min or 128 MB) avoids small files problem
- Parquet conversion in Firehose optimizes Athena query costs
- Athena for SQL queries across all log types
- Partition by date for cost-effective queries

---

## Pattern E: Machine Learning Data Pipeline

```
S3 (raw data) --> Glue ETL (clean, feature engineering) --> S3 (features)
              --> SageMaker (training) --> Model
              --> Bedrock (knowledge base with vector embeddings)
```

### Key Design Decisions
- Glue for data preparation and feature engineering
- Separate S3 buckets/prefixes for raw, processed, and feature data
- Lake Formation for access control on ML datasets

---

## Anti-Patterns (When NOT to Use)

| Anti-Pattern | Why Wrong | Correct Choice |
|-------------|-----------|---------------|
| Lambda for heavy ETL | 15 min timeout, 10GB memory | Glue or EMR |
| RDS for analytics | Row-based storage, slow aggregation | Redshift (columnar) |
| KDS for long-term storage | Expensive retention | Firehose -> S3 |
| Glue Crawlers on every file | Cost and time overhead | Partition projection, event-based updates |
| Small files in S3 | Slow queries, metadata overhead | Firehose buffering, Glue compaction |
| Embedding credentials in code | Security risk | Secrets Manager, IAM Roles |
| Using NAT Gateway for S3 | Unnecessary cost | S3 Gateway Endpoint (free) |

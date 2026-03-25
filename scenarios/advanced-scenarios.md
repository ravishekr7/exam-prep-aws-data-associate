# Advanced Scenarios

Complex exam-style scenarios with constraint analysis.

---

## Scenario 1: Real-Time Fraud Detection

**Requirement:** Banking transactions must be analyzed for fraud patterns. Flag potential fraud within 1-2 seconds.

**Constraint:** Strictly low latency (< 5 seconds)

| Option | Verdict | Why |
|--------|---------|-----|
| Firehose -> S3 -> Athena | WRONG | Firehose min buffer 60s + Athena query time |
| Glue Streaming | WRONG | Cold start overhead, micro-batch not fast enough |
| **KDS -> Flink -> Lambda/SNS** | CORRECT | KDS ingests instantly, Flink detects patterns sub-second |

---

## Scenario 2: Near Real-Time Log Dashboard

**Requirement:** 1 TB/day of web logs. Dashboard within 5-10 minutes. Minimal management. Cost-effective.

**Constraint:** High volume, loose latency, managed

| Option | Verdict | Why |
|--------|---------|-----|
| KDS -> EC2 consumers | WRONG | Too much management overhead |
| Firehose -> Redshift | WRONG | Expensive for just logs |
| **Firehose (5min/128MB buffer, Parquet) -> S3 -> Athena -> QuickSight** | CORRECT | Serverless, cheap, Parquet optimizes queries |

---

## Scenario 3: GDPR Right to be Forgotten

**Requirement:** Delete specific user data from S3 data lake without rewriting entire files.

**Constraint:** Row-level deletes on S3

| Option | Verdict | Why |
|--------|---------|-----|
| Standard Parquet on S3 | WRONG | Must read entire file, filter, rewrite |
| **Apache Iceberg with Glue** | CORRECT | ACID transactions, row-level deletes on S3 |

---

## Scenario 4: Hot vs Cold Data Split

**Requirement:** 10 years of sales data (50 TB). Last 1 year queried daily, last 9 years queried monthly. Budget tight.

**Constraint:** Cost vs performance balance

| Option | Verdict | Why |
|--------|---------|-----|
| All 50TB in Redshift | WRONG | Expensive storage for rarely used data |
| All 50TB in S3 + Athena | WRONG | Daily reports too slow/costly |
| **Redshift (1yr) + Spectrum (9yr in S3)** | CORRECT | Hot data fast, cold data cheap. Transparent joins |

---

## Scenario 5: PII Masking Before Storage

**Requirement:** EU customer data with credit card numbers. Compliance: unencrypted CC must NEVER be written to disk.

**Constraint:** Mask in transit, before S3

| Option | Verdict | Why |
|--------|---------|-----|
| S3 -> Lambda mask -> new S3 | WRONG | Raw data touches S3 briefly |
| **Firehose -> Lambda transform -> S3** | CORRECT | Firehose buffers in memory, Lambda masks, masked data lands in S3 |

---

## Scenario 6: Cross-Account Data Sharing (No Copying)

**Requirement:** Account A owns S3 data. Account B queries with Athena. No data copying.

**Solution:** S3 Bucket Policy (allow B) + KMS Key Policy (allow B) + Glue Catalog Resource Policy

Account B needs: permission to decrypt (KMS) AND permission to read (S3)

---

## Scenario 7: Over-Provisioned DynamoDB

**Situation:** 1000 WCU provisioned for morning spike. Rest of day only 50 WCU. Paying 24/7.

| Solution | When |
|----------|------|
| Auto-Scaling | Variable but somewhat predictable patterns |
| On-Demand | Truly unpredictable/short spikes |
| **Scheduled Scaling** | Exact predictable pattern (scale up 8:55 AM, down 10 AM) |

---

## Scenario 8: Schema Evolution in Streaming

**Requirement:** Upstream database adds columns weekly. Pipeline breaks on schema mismatch.

**Solution:** Glue Schema Registry + Kinesis Data Streams
- Producers register new schema version
- Forward Compatibility mode: consumers handle new columns without crashing

---

## Scenario 9: Redshift Cluster at 100% CPU

**Requirement:** Run massive aggregation without impacting dashboard users.

**Solution:** Redshift Spectrum
- Runs aggregation on Spectrum's own compute fleet
- Main cluster resources stay available for dashboards

---

## Scenario 10: Lightweight File Processing

**Requirement:** Users upload small Excel files sporadically. Validate header, convert to CSV.

**Constraint:** Event-driven, sporadic, lightweight

| Option | Verdict | Why |
|--------|---------|-----|
| Glue Job | WRONG | Min 1 DPU billing, 20s cold start |
| EMR | WRONG | Overkill |
| **S3 Event -> Lambda** | CORRECT | ms startup, pay per ms, perfect for small files |

---

## Scenario 11: Real-Time Fraud Detection Pipeline (End-to-End)

### Business Problem
A fintech company processes 50,000 payment transactions per second globally. They need to detect fraud patterns (e.g., multiple transactions from different geographies within 5 minutes, card used after reported stolen) and flag suspicious transactions in under 2 seconds. Flagged transactions must be stored for 7 years for regulatory compliance.

### AWS Architecture
```
Payment Systems
     │
     ▼
Kinesis Data Streams (10 shards, 50K records/sec)
     │
     ├──▶ Managed Service for Apache Flink
     │         - Stateful CEP (Complex Event Processing)
     │         - 5-minute sliding window per card_id
     │         - Pattern: >3 transactions in different cities within 5 min
     │         - Output: fraud score + flagged transaction record
     │              │
     │              ├──▶ DynamoDB (flagged transactions table)
     │              │         - Partition key: card_id
     │              │         - Sort key: timestamp
     │              │         - TTL: 30 days (active cases)
     │              │         - Low-latency lookup by fraud investigation team
     │              │
     │              └──▶ SNS → Lambda → Block card API call (real-time response)
     │
     └──▶ Kinesis Firehose
               - Buffer: 60 seconds / 128 MB
               - Format: Parquet via Glue schema
               - Destination: S3 (raw archive, 7-year retention via Glacier lifecycle)
```

### Key Service Choices with Justification
- **Kinesis Data Streams:** Required for multiple consumers (Flink + Firehose). Provides replay capability for debugging false positives.
- **Managed Service for Apache Flink:** Only option for stateful CEP with sliding time windows across millions of card IDs. Lambda cannot maintain state across invocations without external storage.
- **DynamoDB:** Sub-millisecond reads for fraud investigators looking up a card's recent suspicious activity. On-Demand mode handles unpredictable spike when a fraud wave occurs.
- **Firehose → S3:** Parallel archiving path. Minimum 60-second latency is fine for compliance archiving (regulatory teams query historical data, not real-time).

### Security Considerations
- KDS and DynamoDB encrypted with KMS CMK. CloudTrail audits every KMS key usage.
- Flink application IAM role: `kinesis:GetRecords`, `kinesis:DescribeStream`, `dynamodb:PutItem` — nothing more.
- S3 archive: SSE-KMS, Object Lock (WORM) for 7 years (regulatory requirement).
- Network: Flink application in VPC, Interface Endpoints for DynamoDB and Kinesis — no public internet.

### Monitoring
- **Flink:** `uptime` metric (is app running?), `numberOfFailedCheckpoints` (data loss risk), `currentOutputWatermark` (processing lag)
- **Kinesis:** `IteratorAgeMilliseconds` (if rising, Flink is falling behind), `GetRecords.IteratorAgeMilliseconds`
- **DynamoDB:** `ThrottledRequests` (GSI capacity), `SuccessfulRequestLatency`
- CloudWatch Alarm: If `numberOfFailedCheckpoints > 0` → SNS → PagerDuty (immediate response)

---

## Scenario 12: GDPR Right-to-Erasure Pipeline

### Business Problem
A European e-commerce company must honor GDPR "Right to Erasure" (Right to be Forgotten) requests within 72 hours. User data exists in: DynamoDB (active user profiles), S3 data lake (event history in Parquet format), and Redshift (analytics).

### AWS Architecture
```
User Deletion Request (via API Gateway)
     │
     ▼
Lambda (orchestration trigger)
     │
     ├──▶ DynamoDB: Set TTL = now (profile expires immediately)
     │
     ├──▶ S3: Tag affected objects with x-amz-delete-marker
     │         Lifecycle rule: delete tagged objects within 24 hours
     │
     ├──▶ Step Functions: Trigger erasure workflow
     │         State 1: Glue Job — Iceberg row-level DELETE for user_id
     │              - DELETE FROM events WHERE user_id = 'user_123'
     │              - Iceberg handles ACID delete without full file rewrite
     │         State 2: Redshift Query — DELETE FROM all tables WHERE user_id = 'user_123'
     │         State 3: Wait 24 hours (DynamoDB TTL + S3 lifecycle to complete)
     │         State 4: Lambda — Trigger Amazon Macie scan on affected S3 paths
     │         State 5: Evaluate Macie findings → if PII found, re-trigger Glue job
     │
     └──▶ CloudTrail: Records all deletion API calls for audit proof
```

### Key Service Choices with Justification
- **DynamoDB TTL:** Zero-WCU deletion of user profile. Happens asynchronously within 48 hours (set TTL = current timestamp to trigger immediately).
- **Apache Iceberg:** Row-level DELETE on S3 without rewriting all Parquet files. Critical for performance on large event history tables (billions of rows). Standard Parquet would require full file rewrites.
- **Amazon Macie:** Automated PII detection to verify erasure was complete. Macie scans S3 and identifies remaining PII patterns (email, SSN, credit card).
- **CloudTrail:** Creates immutable audit trail of all API calls related to the deletion request — required for GDPR compliance proof.
- **Step Functions Standard:** Long-running workflow (up to 24 hours for TTL/lifecycle to complete). Exactly-once semantics ensure deletion steps are not repeated unintentionally.

### Security Considerations
- Deletion Lambda role has minimal permissions: `dynamodb:UpdateItem` (TTL only), `s3:PutObjectTagging`, `glue:StartJobRun`, `redshift-data:ExecuteStatement`.
- CloudTrail logs are written to a separate compliance S3 bucket with Object Lock enabled — tamper-proof audit trail.

---

## Scenario 13: Cross-Account Data Sharing for Analytics

### Business Problem
A media company's data engineering team (Account A) maintains a central data lake with Glue Data Catalog. Marketing analytics (Account B) and Finance (Account C) teams need to query this data via Athena in their own accounts, but marketing should only see campaign data and finance should only see revenue data — not each other's data.

### AWS Architecture
```
Account A (Data Producer):
  S3 Data Lake
  ├── s3://central-lake/campaigns/    ← tagged: dept=marketing
  ├── s3://central-lake/revenue/      ← tagged: dept=finance
  └── s3://central-lake/shared/       ← tagged: dept=all

  AWS Glue Data Catalog
  ├── marketing_db (LF-Tag: dept=marketing)
  └── finance_db   (LF-Tag: dept=finance)

  Lake Formation:
  - LF-Tag: dept=marketing → granted to Account B
  - LF-Tag: dept=finance → granted to Account C
  - Column security: hide PII columns (email, phone) from Account B

  AWS RAM Share:
  - Share marketing_db with Account B
  - Share finance_db with Account C

Account B (Marketing Analytics):
  1. Accept RAM share for marketing_db
  2. Lake Formation grant: SELECT on dept=marketing to AnalystsGroup
  3. Athena queries: SELECT * FROM marketing_db.campaigns
     → Lake Formation filters: only dept=marketing data returned
     → Column filter: PII columns (email) not visible

Account C (Finance):
  1. Accept RAM share for finance_db
  2. Athena queries: SELECT * FROM finance_db.revenue
     → Lake Formation filters: only dept=finance data
```

### Key Service Choices with Justification
- **LF-Tags:** Scale to any number of tables — new marketing tables tagged `dept=marketing` automatically inherit Account B's access. No per-table grants needed.
- **AWS RAM:** Standard mechanism for sharing Glue Catalog resources cross-account. Lake Formation permissions travel with the share.
- **Column-level security in Lake Formation:** Marketing analysts need campaign performance but not individual PII. Lake Formation strips PII columns before query results are returned.
- **Athena in Account B/C:** Queries run in the consumer account, charges for data scanned go to the consumer. The data stays in Account A's S3.

### Security Considerations
- Account A's S3 bucket policy grants read access to Account B and C roles that Athena uses.
- KMS key policy in Account A grants `kms:Decrypt` to Account B and C query roles.
- All cross-account access logged in Account A's CloudTrail.

### Monitoring
- Account A: CloudTrail → Athena → S3: which accounts are querying which tables, how often
- Lake Formation access denied events → CloudWatch Logs → alerts if unauthorized access attempts

---

## Scenario 14: CDC Pipeline from On-Premises to Data Lake

### Business Problem
A retail company has an Oracle 19c database on-premises with 500 GB of customer and transaction data. They need to migrate this data to AWS and maintain near-real-time synchronization (< 5 minutes lag) as new transactions occur. The resulting data lake should be queryable via Athena.

### AWS Architecture
```
On-Premises Oracle DB (500 GB)
     │
     ▼ (Direct Connect or VPN)
AWS DMS Replication Instance
  ├── Phase 1: Full Load
  │      Read all existing data → write to Kinesis Data Streams
  │      (or directly to S3 for large initial load)
  │
  └── Phase 2: CDC (Change Data Capture)
         Oracle LogMiner reads redo logs
         INSERT/UPDATE/DELETE events → Kinesis Data Streams
              │
              ▼
         AWS Lambda (light transformation)
           - Add metadata: source_system, ingestion_timestamp
           - Normalize column names (Oracle UPPERCASE → snake_case)
           - Filter out internal Oracle system tables
              │
              ▼
         S3 Landing Zone
           Format: Apache Iceberg (via Glue job)
           s3://datalake/landing/customers/
           s3://datalake/landing/transactions/
              │
              ▼
         AWS Glue ETL Job (hourly)
           - Read Iceberg landing zone
           - Apply SCD Type 2 logic for customer dimension
           - Write to Iceberg curated zone
              │
              ▼
         Glue Crawler → AWS Glue Data Catalog → Athena queries
         (auto-discovers new Iceberg partitions)
```

### Key Service Choices with Justification
- **AWS DMS:** Purpose-built for heterogeneous database migration. Oracle → Kinesis is a supported DMS target. Native CDC via Oracle LogMiner — no application changes required.
- **Kinesis Data Streams (not Firehose):** DMS CDC produces a continuous, low-volume stream. Kinesis allows multiple consumers (Lambda + monitoring). Firehose would add 60-second minimum latency.
- **Lambda for light transformation:** Per-record metadata enrichment. Stateless, sub-100ms per record. Too heavy for Glue (cold start) or Flink (overkill for simple field mapping).
- **Apache Iceberg:** Handles UPSERTS from CDC natively (MERGE INTO). Oracle UPDATEs translate to Iceberg MERGE operations — update specific rows without rewriting entire files.
- **Glue Crawler:** Auto-discovers new Iceberg table partitions as data arrives. Without crawler, Athena doesn't know new partitions exist.

### Security Considerations
- DMS replication instance in a private subnet. Direct Connect or VPN for on-premises connectivity.
- DMS task: source Oracle credentials in Secrets Manager, rotated automatically.
- S3 landing zone: SSE-KMS. Only DMS replication instance role and Glue ETL role have write access.
- Athena: Lake Formation column-level security to hide PII columns (SSN, credit card) from analyst roles.

### Monitoring
- **DMS:** `CDCLatencySource` (lag in reading Oracle redo logs), `CDCLatencyTarget` (lag in writing to Kinesis). If either rises above 5 minutes → alarm.
- **Kinesis:** `IteratorAgeMilliseconds` — if Lambda is falling behind.
- **Glue Crawler:** Last crawl time + `TablesCreated` / `TablesUpdated` metrics.
- CloudWatch Dashboard showing end-to-end latency from Oracle commit to Athena queryable.

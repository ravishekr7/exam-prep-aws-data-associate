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

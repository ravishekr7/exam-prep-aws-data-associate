# Amazon S3 - Ingestion Patterns

S3 is the foundation of the AWS Data Lake. This covers ingestion-specific patterns.

---

## Ingestion Performance Optimization

### 1. Multipart Upload
- **Required** for objects > 5GB
- **Recommended** for objects > 100MB
- Uploads parts in parallel; if one fails, only retry that part
- Can pause and resume uploads

### 2. S3 Transfer Acceleration
- Uses AWS Edge Locations
- Upload to nearest edge location -> internal AWS backbone -> target bucket
- Best for global/cross-continent uploads
- Must be enabled on the bucket

### 3. Request Rate Performance
- S3 scales automatically to **3,500 PUT/COPY/POST/DELETE** and **5,500 GET** requests per second per prefix
- Randomized prefixes (legacy optimization) rarely needed post-2018
- Still relevant at extreme scale

---

## Event Notifications

Trigger actions when objects are created or deleted.

**Triggers:** ObjectCreated, ObjectRemoved, ObjectRestore, Replication

**Destinations:**
- Lambda
- SQS
- SNS
- EventBridge (for advanced routing)

**Pattern:** S3 Event -> SQS -> Lambda (fan-out / buffer) -> Processing

---

## S3 Select

- Retrieve a **subset** of data from S3 using SQL expressions
- Saves bandwidth and processing time
- Works with CSV, JSON, and Parquet
- Use for filtering before full processing

---

## Batch Ingestion Tools

| Tool | Use Case |
|------|----------|
| **AWS Transfer Family** | SFTP, FTPS, FTP directly to S3 |
| **AWS Snow Family** | Petabyte/Exabyte scale offline transfer (Snowball Edge, Snowmobile) |
| **AWS DataSync** | Online data transfer from on-prem NFS/SMB to S3 |
| **Glue Crawlers/Jobs** | Scheduled ingestion runs |
| **Amazon AppFlow** | SaaS integrations (Salesforce, SAP, etc.) to S3 |

---

## Exam Gotchas

- **Multipart upload** is the answer when asked about uploading large files efficiently
- **Transfer Acceleration** is the answer for "global users uploading to a single bucket"
- **S3 Event Notifications** can go to Lambda, SQS, SNS, or EventBridge
- S3 Select reduces the amount of data transferred - useful for "minimize cost" scenarios
- **AppFlow** is the answer for SaaS-to-AWS data ingestion without custom code

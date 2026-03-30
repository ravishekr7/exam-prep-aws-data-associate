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
| **Glacier Flexible Retrieval** | Archive | 1 min – 12 hrs (tier-dependent) | 90 days | Backups, compliance data |
| **Glacier Deep Archive** | Rare archive | 12-48 hours | 180 days | Regulatory archives, lowest cost |

### Intelligent-Tiering (Exam Favorite)
- Moves data between frequent/infrequent access tiers automatically
- No retrieval fees
- Small monitoring fee per object
- Best default choice when access patterns are unknown

### Glacier Flexible Retrieval — Retrieval Tiers
Glacier Flexible Retrieval has three retrieval speed options with different costs:

| Tier | Retrieval Time | Cost | Use Case |
|---|---|---|---|
| **Expedited** | 1–5 minutes | Highest per-GB | Occasional urgent access |
| **Standard** | 3–5 hours | Medium | Routine restores |
| **Bulk** | 5–12 hours | Lowest | Large-scale, non-urgent restores |

Exam pattern: "Restore from Glacier within 4 hours at lowest cost" → Glacier Flexible Retrieval, Standard tier. "Must restore within 5 minutes" → Expedited tier (or Glacier Instant Retrieval if instant-ms access needed regularly).

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

### File Formats

Choosing the right file format is one of the most impactful data lake decisions for query performance and cost.

| Format | Orientation | Compression | Best For | Avoid For |
|---|---|---|---|---|
| **Parquet** | Columnar | Snappy, GZIP, ZSTD | Athena, Redshift Spectrum, Glue analytics | Streaming row-by-row writes |
| **ORC** | Columnar | Zlib, Snappy | Hive on EMR, Hive Metastore workloads | Non-Hive environments |
| **Avro** | Row | Snappy, Deflate | Streaming ingest (Kafka/Kinesis), schema evolution | Large-scale analytics reads |
| **CSV/JSON** | Row | None by default | Raw landing zone, small files, human-readable | Any large-scale analytical query |

**Why columnar formats win for analytics:**
- A query `SELECT revenue FROM sales` on a Parquet file reads only the `revenue` column bytes — not the entire row
- On CSV, the same query reads every column of every row, then discards everything except `revenue`
- For a 100-column table where you query 3 columns, Parquet reads ~97% less data → 97% cheaper Athena query

**Exam pattern:** "Convert raw CSV files in S3 to a format that reduces Athena query cost and time" → Convert to Parquet using a Glue ETL job. Parquet is the default recommendation for Athena and Redshift Spectrum.

### Small Files Problem

**Symptom:** Thousands of small files (< 128 MB each) in S3 cause slow Athena/Spark/Glue queries even on small total data volumes.

**Why it happens:** Each file requires a separate S3 API call and a separate processing task. 10,000 × 10 KB files = 10,000 task launches with near-zero work per task — all overhead, no throughput.

**Common causes:**
- Kinesis Firehose writing many small buffered files
- Streaming jobs flushing too frequently
- Many small Glue/Spark output partitions

**Fixes:**
- **Glue ETL job:** Set `groupFiles=inPartition` and `groupSize` to coalesce small files on read, then write as fewer large files
- **S3 Lifecycle + Rewrite job:** Periodically run a Glue/Spark job that reads small files and writes them back as consolidated Parquet
- **Kinesis Firehose:** Increase buffer size (up to 128 MB) and buffer interval (up to 900 seconds) to produce larger files

**Exam pattern:** "Athena queries on a Kinesis Firehose → S3 pipeline are slow despite small data volumes" → Small files problem — increase Firehose buffer size or add a Glue compaction job.

### Data Lake Zones
Standard architecture pattern tested in exam scenarios:

| Zone | Also Called | Contents | Storage Class |
|---|---|---|---|
| **Raw** | Landing, Bronze | Unmodified source data as-is | Standard or Intelligent-Tiering |
| **Curated** | Processed, Silver | Cleaned, validated, converted to Parquet | Standard or Standard-IA |
| **Consumption** | Refined, Gold | Aggregated, business-ready datasets | Standard |

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

### Lifecycle Rules for Versioned Buckets

When versioning is enabled, deleted and overwritten objects become **non-current versions** and continue to accrue storage charges indefinitely unless lifecycle rules manage them.

Two additional rule types for versioned buckets:

| Rule Type | What It Does |
|---|---|
| **Non-current version expiration** | Permanently delete non-current versions after N days |
| **Expired object delete marker cleanup** | Remove delete markers left behind after all versions are gone |

Example: Delete non-current versions after 30 days to prevent unbounded cost growth:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-versioned-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    }]
  }'
```

**Exam pattern:** "S3 costs keep growing after enabling versioning even though no new data is being added" → Non-current versions are accumulating — add a non-current version expiration lifecycle rule.

---

## S3 Versioning

- Keeps all versions of an object
- Protects against accidental deletion (delete marker instead of permanent delete)
- Combined with MFA Delete for extra protection
- Required for Cross-Region Replication (CRR) and Same-Region Replication (SRR)
- **Cost trap:** Every version of every object is stored and billed — add non-current version expiration lifecycle rules to control cost

---

## S3 Replication (CRR and SRR)

### Cross-Region Replication (CRR)
- Replicate objects to a bucket in a **different region**
- Use for: disaster recovery, compliance (data sovereignty), reducing read latency for cross-region consumers
- Requires versioning enabled on both source and destination buckets

### Same-Region Replication (SRR)
- Replicate objects to a bucket in the **same region**
- Use for: log aggregation into a central account, live copy between production and test accounts, data sovereignty requirements within a region

### Critical Replication Constraints (Exam Traps)

**Only new objects are replicated by default.**
Enabling CRR or SRR does not replicate objects already in the source bucket. To replicate existing objects, use **S3 Batch Replication** — a separate Batch Operations job that processes existing objects on demand.

**Delete markers are not replicated by default.**
When you delete an object in the source (creating a delete marker), the deletion is not propagated to the destination. Objects deleted in the source still appear in the destination unless "Delete marker replication" is explicitly enabled in the replication rule.

**What is not replicated:**
- Objects in Glacier or Glacier Deep Archive (already archived, not replicated)
- Objects encrypted with SSE-C (customer-provided keys — key material is not available to S3)
- Objects that are themselves replicas (no chaining of replications by default)

### S3 Replication Time Control (RTC)
- Add-on for CRR/SRR that guarantees 99.99% of objects are replicated within **15 minutes**
- Provides replication metrics and CloudWatch alerts for replication latency
- Additional cost over standard replication

**Exam pattern:** "Replicate S3 objects to another region with a guaranteed SLA of 15 minutes" → CRR + Replication Time Control (RTC).

### Replication Exam Patterns
- "After enabling CRR, existing objects are not appearing in the destination" → CRR only replicates new objects; use S3 Batch Replication for existing objects
- "Deleted objects in source still appear in destination bucket" → Delete marker replication is not enabled
- "Aggregate logs from multiple accounts into one S3 bucket in the same region" → SRR (not CRR — same region)

---

## S3 Bucket Keys for SSE-KMS

When SSE-KMS is enabled on a bucket, every S3 PUT and GET operation calls the KMS API to generate or retrieve a data key. At high throughput (thousands of operations/second), this causes KMS API throttling errors even within KMS's own quotas.

S3 Bucket Keys solve this by generating a short-lived bucket-level data key from KMS, which is then used locally to encrypt/decrypt individual objects — without calling KMS for each object.

**Effect:** KMS API calls reduced by up to 99%. Cost also reduced significantly (KMS charges per API call).

Enable at bucket level:

```bash
aws s3api put-bucket-encryption \
  --bucket my-data-lake-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123:key/abc-def"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

Or at object level during upload:

```bash
aws s3 cp myfile.parquet s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:... \
  --bucket-key-enabled
```

**Exam Patterns:**
- "SSE-KMS is causing KMS throttling errors on a high-throughput S3 bucket" → Enable S3 Bucket Keys
- "Reduce KMS costs for a data lake bucket receiving millions of objects per day" → S3 Bucket Keys
- "Encrypt all objects with a customer-managed KMS key without KMS rate limit issues" → SSE-KMS + S3 Bucket Keys enabled

---

## S3 Encryption Types

| Type | Key Managed By | Use Case |
|---|---|---|
| **SSE-S3** | S3 (AWS managed, opaque) | Default encryption, no key management needed |
| **SSE-KMS** | KMS (customer-managed or AWS-managed CMK) | Audit trail, key rotation, cross-account access control |
| **SSE-C** | Customer (key passed per request) | Customer retains full key control; key never stored in AWS |
| **Client-side** | Customer (encrypt before upload) | End-to-end encryption; S3 never sees plaintext |

**Exam traps:**
- SSE-C objects cannot be replicated via CRR/SRR (AWS doesn't have the key material)
- SSE-KMS requires `kms:GenerateDataKey` and `kms:Decrypt` in the caller's IAM policy — just `s3:PutObject` alone is insufficient
- SSE-S3 uses AES-256 but provides no audit trail in CloudTrail for individual object decrypt events (SSE-KMS does)

---

## S3 Object Lock

Prevents objects from being deleted or overwritten for a fixed period or indefinitely. Implements **WORM (Write Once Read Many)** storage.

**Two retention modes:**

| Mode | Who Can Override | Use Case |
|---|---|---|
| **Governance** | Users with `s3:BypassGovernanceRetention` IAM permission | Testing retention policies, flexibility for admins |
| **Compliance** | No one — including the root account — until retention expires | Strict regulatory requirements (SEC 17a-4, FINRA, HIPAA) |

**Legal Hold:** An indefinite lock with no retention period. Any user with `s3:PutObjectLegalHold` can apply or remove it independently of the retention mode. Useful when litigation hold is needed with no fixed end date.

**Exam patterns:**
- "Regulatory requirement — objects must not be deletable by anyone, including root, for 7 years" → Object Lock in **Compliance** mode
- "Need WORM but want admins to be able to override for testing purposes" → Object Lock in **Governance** mode
- "Legal team requests an indefinite hold on specific objects with no known end date" → Legal Hold

---

## S3 Event Notifications

S3 Event Notifications trigger a downstream service whenever an object event occurs in a bucket.

**Supported event types:**
- `s3:ObjectCreated:*` (all create events: Put, Post, Copy, CompleteMultipartUpload)
- `s3:ObjectCreated:Put` (only PutObject)
- `s3:ObjectRemoved:*` (Delete, DeleteMarkerCreated)
- `s3:ObjectRestore:Post` (Glacier restore initiated)
- `s3:ObjectRestore:Completed` (Glacier restore ready)
- `s3:Replication:*` (replication failure events)

**Destinations (native S3 notifications):**
- AWS Lambda (invoked **asynchronously** — S3 does not wait for a response)
- Amazon SQS queue (add message to queue)
- Amazon SNS topic (fan-out to subscribers)

**EventBridge vs Native S3 Notifications:**

| Feature | Native S3 Notifications | S3 → EventBridge |
|---|---|---|
| Destinations | Lambda, SQS, SNS only | 20+ AWS services |
| Filtering | Prefix and suffix only | Rich content-based filtering (size, metadata, tags) |
| Multiple targets | One per event type | Many rules, many targets |
| Archive/replay | No | Yes (EventBridge archive) |
| Setup | Per-bucket notification config | One setting: "Send to EventBridge" |

When to use EventBridge: You need multiple targets per event, content-based filtering, or routing to services beyond Lambda/SQS/SNS.

**Common data pipeline pattern:**
```
S3 (new Parquet file lands) → EventBridge → Step Functions (start pipeline)
                                           → Lambda (update partition in Glue Catalog)
                                           → SNS (alert data team)
```

**Exam Patterns:**
- "Trigger a Glue crawler when a new file lands in S3 with least custom code" → S3 Event Notification → EventBridge → Glue StartCrawler API
- "Route S3 events to multiple downstream services with content-based filtering" → Enable S3 → EventBridge integration, create EventBridge rules
- "Notify multiple SQS queues when an object is created in S3" → S3 → SNS (fan-out) → multiple SQS queues

---

## S3 Select and Glacier Select

S3 Select allows applications to retrieve a subset of data from an S3 object using SQL expressions — the filtering happens **server-side inside S3**, so only matching data is transferred over the network.

**How it works:**
```
Without S3 Select: S3 → download entire 10 GB Parquet file → filter in application → use 50 MB result
With S3 Select:    S3 → evaluate WHERE clause inside S3   → return 50 MB result only
```

**Supported formats:** CSV, JSON, Parquet (and GZIP/BZIP2 compressed CSV/JSON)

```python
import boto3

s3 = boto3.client('s3')

response = s3.select_object_content(
    Bucket='my-data-lake',
    Key='sales/2024/q4.csv',
    ExpressionType='SQL',
    Expression="SELECT * FROM s3object WHERE region = 'APAC' AND revenue > 10000",
    InputSerialization={'CSV': {'FileHeaderInfo': 'USE'}},
    OutputSerialization={'CSV': {}}
)
```

**Glacier Select:** Same concept applied to objects in Glacier Flexible Retrieval — filter before restoring, reducing restore cost and time.

**Exam Patterns:**
- "Retrieve only rows matching a filter from a large S3 CSV file — minimize data transfer cost" → S3 Select
- "An application downloads entire S3 objects but only uses 5% of the data — how to reduce transfer cost?" → S3 Select (push the filter to S3)
- "Filter Glacier archive contents without fully restoring the object" → Glacier Select

---

## S3 Access Points

S3 Access Points are named network endpoints attached to a bucket, each with their own access policy. They simplify access management for shared data lake buckets used by multiple teams.

**The problem they solve:**
A single S3 bucket policy for a shared data lake becomes a massive, complex document managing permissions for dozens of teams, applications, and services. Any change risks breaking another team's access.

**How Access Points work:**
- Each team or application gets their own access point (named endpoint + policy)
- The access point policy scopes what that consumer can access within the bucket
- Up to 10,000 access points per bucket
- Can be restricted to a specific VPC (no public internet access)

```bash
# Create an access point for the Finance team, restricted to their VPC
aws s3control create-access-point \
  --account-id 123456789012 \
  --name finance-team-ap \
  --bucket my-data-lake-bucket \
  --vpc-configuration VpcId=vpc-abc123
```

Access point ARN used in IAM policies:
```
arn:aws:s3:us-east-1:123456789012:accesspoint/finance-team-ap
```

**Exam Patterns:**
- "Multiple teams share one S3 data lake bucket — manage each team's access without a complex monolithic bucket policy" → S3 Access Points (one access point per team)
- "Restrict a team's S3 access to within their VPC only" → Access Point with VPC configuration
- "Simplify permission management as the number of data consumers grows" → S3 Access Points (scales to 10,000 per bucket)

---

## S3 Presigned URLs

A presigned URL grants temporary access to a specific S3 object (GET or PUT) without requiring the recipient to have AWS credentials or bucket permissions.

**How it works:** The URL contains a cryptographic signature generated using the creator's credentials. Anyone with the URL can access the object until the URL expires.

**Generate a presigned URL for download (GET):**

```python
import boto3

s3 = boto3.client('s3')

url = s3.generate_presigned_url(
    ClientMethod='get_object',
    Params={
        'Bucket': 'my-data-bucket',
        'Key': 'reports/q4-2024.pdf'
    },
    ExpiresIn=3600  # 1 hour in seconds (max 7 days = 604800 seconds for IAM user credentials)
)
# url is a regular HTTPS URL — share it with anyone
```

**Generate a presigned URL for upload (PUT):**

```python
url = s3.generate_presigned_url(
    ClientMethod='put_object',
    Params={
        'Bucket': 'my-upload-bucket',
        'Key': 'uploads/user123/data.csv',
        'ContentType': 'text/csv'
    },
    ExpiresIn=900  # 15 minutes
)
# Recipient can HTTP PUT to this URL to upload directly to S3
```

**Key facts:**
- Maximum expiry: 604800 seconds (7 days) — **only when signed with long-term IAM user credentials**
- **STS/role credentials cap expiry at the session token lifetime.** If you generate a presigned URL using a role (EC2 instance profile, Lambda execution role, assumed role), the URL expires at whichever is shorter: `ExpiresIn` or the STS session duration. A 7-day URL signed with a 1-hour role token expires in 1 hour.
- The URL uses the permissions of the IAM entity that created it
- If the creating entity's permissions are revoked, the URL stops working immediately
- No changes to bucket policy or IAM needed

**Exam Patterns:**
- "Provide a partner company temporary read access to a specific S3 object without modifying IAM or bucket policy" → Presigned URL
- "Allow mobile app users to upload files directly to S3 without going through your API server" → Presigned URL for PUT (reduces server load, data goes directly to S3)
- "Share a private S3 report link that expires after 24 hours" → Presigned URL with `ExpiresIn=86400`
- "Presigned URL generated from a Lambda function expires after 1 hour despite setting ExpiresIn to 7 days" → Lambda execution role STS token caps the URL expiry at the session duration

---

## S3 Transfer Acceleration

S3 Transfer Acceleration speeds up uploads to S3 from distant geographic locations by routing data through Amazon CloudFront's globally distributed edge network.

**How it works:**
- Client uploads to the nearest CloudFront edge location (fast local network hop)
- CloudFront routes data over AWS's optimized backbone network to the S3 bucket region
- Much faster than routing over the public internet across continents
- Also benefits downloads from distant locations (not just uploads)

**Enable on a bucket:**

```bash
aws s3api put-bucket-accelerate-configuration \
  --bucket my-global-data-bucket \
  --accelerate-configuration Status=Enabled
```

**Use the accelerate endpoint:**

```
# Standard endpoint:
my-bucket.s3.us-east-1.amazonaws.com

# Accelerate endpoint:
my-bucket.s3-accelerate.amazonaws.com
```

Additional cost: per GB transferred through acceleration. Only worth it for large files from distant locations — use the S3 Transfer Acceleration speed comparison tool to verify benefit before enabling.

**Exam Patterns:**
- "Data is uploaded from offices in Asia and South America to a US S3 bucket and uploads are slow" → S3 Transfer Acceleration
- "Fastest way to upload large files to S3 from globally distributed locations" → Transfer Acceleration (not multipart alone)
- "Reduce upload latency for a global IoT fleet sending data to a single-region S3 bucket" → Transfer Acceleration

---

## Multipart Upload — Key Details

Multipart upload splits large objects into parts that are uploaded independently and in parallel, then assembled in S3.

**Requirements and limits:**
| Limit | Value |
|---|---|
| Minimum part size | 5 MB (except last part) |
| Maximum part size | 5 GB |
| Maximum object size | 5 TB (multipart required for anything > 5 GB) |
| Recommended threshold | Use multipart for objects > 100 MB |
| Maximum parts per object | 10,000 |

**Benefits:**
- Resume failed uploads — only re-upload the failed part, not the whole object
- Parallel part uploads = faster throughput for large objects
- Begin uploading parts before the full object is finished being created

**The hidden cost trap — incomplete multipart uploads:**

When a multipart upload is initiated but never completed (e.g., upload fails and the client doesn't call `AbortMultipartUpload`), the uploaded parts accumulate in S3 and incur storage charges — even though no complete object exists and nothing appears in the S3 console.

Fix: Create a lifecycle rule to automatically abort incomplete multipart uploads:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "abort-incomplete-mpu",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }]
  }'
```

**Exam Patterns:**
- "S3 storage costs are unexpectedly high despite no complete objects being visible" → Incomplete multipart uploads accumulating — add lifecycle rule to abort them
- "Upload a 50 GB file to S3 — what must you use?" → Multipart upload (mandatory for > 5 GB, recommended for > 100 MB)
- "Resume a failed large file upload without starting over" → Multipart upload (re-upload only the failed part)

---

## Exam Gotchas

- **Intelligent-Tiering** = answer when "access patterns are unknown" or "minimize retrieval cost risk"
- **Lifecycle policies** = "automate cost optimization" or "move data to cheaper storage over time"
- Glacier Deep Archive = cheapest but 12-48 hour retrieval (not for any real-time needs)
- One Zone-IA = only for re-creatable data (less durable)
- S3 requests are per-prefix: 3,500 PUT and 5,500 GET per second per prefix
- **S3 Object Lock Compliance mode** = no one (including root) can delete until retention expires. Governance mode = admins with `s3:BypassGovernanceRetention` can override.
- **CRR/SRR only replicates new objects.** Existing objects require S3 Batch Replication. Enabling replication after the fact does nothing for historical data.
- **Delete markers are not replicated by default.** Objects deleted in source still appear in destination unless delete marker replication is explicitly enabled.
- **Parquet is the default answer for "reduce Athena query cost/time."** Converting CSV/JSON to Parquet is the standard data lake optimization — columnar format reads only needed columns.
- **Small files kill Athena performance.** Many files < 128 MB cause overhead. Fix: increase Firehose buffer size or run Glue compaction jobs.
- **S3 Bucket Keys must be explicitly enabled — SSE-KMS alone does not reduce KMS calls.** If a scenario describes KMS throttling on S3, Bucket Keys is the specific fix. Without Bucket Keys, every object PUT/GET triggers a KMS GenerateDataKey or Decrypt API call.
- **Presigned URLs signed with role/STS credentials expire at the shorter of `ExpiresIn` or the STS session duration** — not necessarily 7 days. A Lambda-generated presigned URL is capped by the Lambda execution role's session token.
- **S3 Event Notifications invoke Lambda asynchronously** — S3 does not wait for Lambda to respond. Use EventBridge for multiple targets or content-based filtering.
- **Incomplete multipart uploads are invisible in the S3 console** (they don't appear as objects) but they do cost money. Always add a lifecycle rule to abort them.
- **Non-current version costs accumulate silently when versioning is on.** Add non-current version expiration lifecycle rules or costs grow indefinitely.
- **SSE-C objects cannot be replicated** — AWS doesn't store the key material, so it cannot re-encrypt objects during replication.
- **Transfer Acceleration adds cost.** Only use it when the distance-related latency is measurably hurting upload performance. AWS provides a speed comparison tool to verify it's beneficial for your use case before enabling.

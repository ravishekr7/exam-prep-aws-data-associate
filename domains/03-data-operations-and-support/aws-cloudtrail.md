# AWS CloudTrail

Auditing service that records WHO did WHAT API calls and WHEN in your AWS account.

---

## Event Types

| Type | What It Captures | Enabled by Default | Cost |
|------|-----------------|-------------------|------|
| **Management Events** | Control plane: CreateBucket, RunJob, CreateUser, DeleteTable | Yes (first copy free) | Free for first trail |
| **Data Events** | Data plane: S3 GetObject/PutObject, Lambda Invoke, DynamoDB GetItem | No | Extra cost per 100K events |
| **Insights Events** | Unusual API call rate or error rate anomalies | No | Extra cost |
| **Network Activity Events** | VPC endpoint API calls | No | Extra cost |

> **Exam tip:** Data Events are NOT enabled by default. You must explicitly enable them. Common exam scenario: "Find who deleted an S3 object" → must have S3 Data Events enabled.

---

## Trails

### Trail Types

| Type | Scope | Use Case |
|------|-------|---------|
| **Single-region trail** | One region only | Region-specific audit |
| **Multi-region trail** | All regions (recommended) | Comprehensive audit — catches global services (IAM, STS, Route 53) |
| **Organization trail** | All accounts in AWS Organization | Centralized auditing for multi-account setups |

### Organization Trail

- Created in the **management account** of an AWS Organization
- Automatically applies to all member accounts — no per-account setup needed
- Member accounts can see the trail but **cannot modify or delete** it
- All events from all accounts delivered to a central S3 bucket in the management account

### Trail Destination

- CloudTrail delivers logs to **S3** (primary) and optionally to **CloudWatch Logs**
- S3 prefix: `s3://bucket/AWSLogs/<account-id>/CloudTrail/<region>/YYYY/MM/DD/`
- Log files are delivered within 15 minutes of the API call (approximate)

---

## Data Events Configuration

Enable data events selectively to control cost:

### S3 Data Events

```
Options:
- All S3 buckets (expensive at scale)
- Specific buckets: arn:aws:s3:::my-bucket/
- Specific prefix: arn:aws:s3:::my-bucket/sensitive-prefix/
- Read events only (GetObject)
- Write events only (PutObject, DeleteObject)
- Read + Write
```

### Lambda Data Events

- Records every `Invoke` API call (who invoked which function, when)
- Enable for specific functions or all functions
- Captures: function ARN, invoking principal, invocation type

### DynamoDB Data Events

- Records `GetItem`, `PutItem`, `DeleteItem`, `Query`, `Scan` at item level
- Enable for specific tables or all tables
- High volume on busy tables — be selective

---

## CloudTrail Insights

Detects **anomalous API activity** compared to the account's historical baseline.

### What It Detects

| Insight Type | Example |
|-------------|---------|
| **API Call Rate** | `CreateUser` suddenly called 50x normal rate (potential credential compromise) |
| **API Error Rate** | `AccessDenied` errors spike (potential enumeration attack) |

### How It Works

1. CloudTrail establishes a **7-day baseline** of normal API activity per service
2. Continuously monitors for deviations (> 2 standard deviations from baseline)
3. Generates an Insights Event when anomaly is detected
4. Generates a second event when activity returns to normal
5. Insights Events delivered to the same S3 bucket as regular events

### Responding to Insights

```
CloudTrail Insights Event → EventBridge Rule → Lambda (investigate + alert)
                                             → SNS (notify security team)
```

---

## CloudTrail Lake

Managed event data lake for CloudTrail — query events directly with SQL, no S3 + Athena setup needed.

### Key Features

| Feature | Details |
|---------|---------|
| **Retention** | 7 years (configurable) |
| **Storage** | Managed by AWS (no S3 buckets to configure) |
| **Query engine** | SQL — similar to Athena |
| **Integration** | EventBridge for real-time event routing |
| **Multi-source** | CloudTrail events + AWS Config data + non-AWS activity |

### CloudTrail Lake vs S3 + Athena

| Attribute | CloudTrail Lake | S3 + Athena |
|-----------|---------------|-------------|
| **Setup** | One click | Trail + S3 + Glue catalog + Athena |
| **Query** | SQL in CloudTrail console | Athena console or API |
| **Cost** | Per GB ingested + scanned | S3 storage + Athena $5/TB scanned |
| **Retention** | Up to 7 years, managed | Configure S3 lifecycle yourself |
| **Cross-account** | Yes (event data stores) | Requires S3 bucket policy + separate trail |
| **Best for** | Governance/compliance teams | Engineering teams integrating with other tools |

### CloudTrail Lake SQL Query Examples

```sql
-- Who deleted an S3 bucket?
SELECT userIdentity.arn, eventTime, requestParameters
FROM <event-data-store-id>
WHERE eventName = 'DeleteBucket'
  AND eventTime > '2024-01-01 00:00:00'
ORDER BY eventTime DESC;

-- All actions by a specific IAM user
SELECT eventName, eventSource, eventTime, sourceIPAddress
FROM <event-data-store-id>
WHERE userIdentity.arn LIKE '%arn:aws:iam::123456789:user/alice%'
  AND eventTime > '2024-01-01'
ORDER BY eventTime DESC;

-- Failed authentication attempts
SELECT userIdentity.arn, eventTime, errorCode, errorMessage, sourceIPAddress
FROM <event-data-store-id>
WHERE errorCode IN ('AccessDenied', 'UnauthorizedOperation', 'InvalidClientTokenId')
  AND eventTime > '2024-01-01'
ORDER BY eventTime DESC;

-- Most-used API calls in the last 30 days
SELECT eventName, eventSource, COUNT(*) AS call_count
FROM <event-data-store-id>
WHERE eventTime > '2024-01-01'
GROUP BY eventName, eventSource
ORDER BY call_count DESC
LIMIT 20;
```

---

## Querying CloudTrail Logs with Athena

When logs are stored in S3, query them with Athena for cost-effective historical analysis.

### Create Athena Table for CloudTrail

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion      STRING,
  userIdentity      STRUCT<
    type:           STRING,
    principalId:    STRING,
    arn:            STRING,
    accountId:      STRING,
    userName:       STRING
  >,
  eventTime         STRING,
  eventSource       STRING,
  eventName         STRING,
  awsRegion         STRING,
  sourceIPAddress   STRING,
  errorCode         STRING,
  errorMessage      STRING,
  requestParameters STRING,
  responseElements  STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/123456789/CloudTrail/us-east-1/';
```

### Common Athena Queries

```sql
-- Who deleted an S3 object?
SELECT userIdentity.arn, eventTime, requestParameters
FROM cloudtrail_logs
WHERE eventName = 'DeleteObject'
  AND eventTime LIKE '2024-01%'
ORDER BY eventTime DESC;

-- IAM changes in the last month
SELECT eventName, userIdentity.arn, eventTime
FROM cloudtrail_logs
WHERE eventSource = 'iam.amazonaws.com'
  AND eventName IN ('CreateUser','DeleteUser','AttachUserPolicy','CreateAccessKey')
ORDER BY eventTime DESC;

-- Root account usage (should be rare)
SELECT eventTime, eventName, sourceIPAddress
FROM cloudtrail_logs
WHERE userIdentity.type = 'Root'
ORDER BY eventTime DESC;
```

---

## Security Best Practices

### Log File Integrity Validation

- CloudTrail generates a **digest file** every hour with SHA-256 hashes of all log files
- Digest files are signed with an AWS private key
- Use `aws cloudtrail validate-logs` CLI command to verify no logs were tampered with or deleted
- Enable with: `--enable-log-file-validation` when creating a trail

### Encryption

- By default: CloudTrail logs encrypted with **SSE-S3** (free)
- For compliance: encrypt with **SSE-KMS** using a customer-managed key (CMK)
- Advantage: KMS provides key usage audit trail in CloudTrail itself
- Grant CloudTrail `kms:GenerateDataKey` permission on the CMK

### Protect the Trail S3 Bucket

```
Protections:
1. S3 bucket policy: deny DeleteObject and PutBucketPolicy except from authorized roles
2. MFA Delete on the bucket: requires MFA to delete objects or change versioning state
3. S3 Object Lock (WORM): compliance mode — no one can delete logs for the retention period
4. S3 Versioning: recover accidentally deleted/overwritten log files
5. Block public access: always on for CloudTrail buckets
```

### Prevent Trail Modification

```json
// SCP to prevent CloudTrail deletion or stopping
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:DeleteTrail",
    "cloudtrail:StopLogging",
    "cloudtrail:UpdateTrail"
  ],
  "Resource": "*"
}
```

Apply this as a Service Control Policy (SCP) at the Organization level.

---

## Common Patterns

### Pattern 1: Compliance Audit Trail

```
Multi-region trail → S3 (SSE-KMS + MFA Delete + Object Lock)
                   → CloudWatch Logs (for real-time alerting)
```

### Pattern 2: Real-Time Security Alerts

```
CloudTrail → CloudWatch Logs → Metric Filter (Root login / IAM change)
                             → CloudWatch Alarm → SNS → PagerDuty
```

### Pattern 3: Forensic Investigation

```
CloudTrail logs in S3 → Athena → SQL queries to trace API calls by principal/resource/time
```

### Pattern 4: Organization-Wide Centralized Logging

```
Management account: Organization trail (covers all member accounts)
                   → Central S3 bucket (logging account)
                   → Athena for cross-account queries
```

### Pattern 5: Anomaly Detection

```
CloudTrail Insights → EventBridge → Lambda (enriches alert with context) → SNS → Security team
```

---

## Key Use Cases (Exam Scenarios)

| Question | Answer |
|----------|--------|
| "Who deleted this S3 object?" | Enable S3 **Data Events** + query CloudTrail |
| "What API calls were made last night?" | CloudTrail Management Events → Athena |
| "Detect unusual API activity automatically" | **CloudTrail Insights** |
| "Query CloudTrail without setting up Athena" | **CloudTrail Lake** |
| "Centralize audit logs across all AWS accounts" | **Organization Trail** |
| "Prove log files were not tampered with" | **Log file integrity validation** |
| "Audit trail for compliance, 7-year retention" | CloudTrail Lake or S3 + Glacier + Object Lock |
| "Who made IAM changes?" | CloudTrail Management Events (IAM is global — use multi-region trail) |

---

## Exam Gotchas

- **Data Events NOT enabled by default** — must opt in; extra cost
- **Management Events free** for the first copy per region per trail
- **CloudTrail is for API auditing** — CloudWatch is for performance monitoring (different purposes)
- **Organization trail** → member accounts cannot modify it (immutable from members' perspective)
- **Log file integrity validation** → proves tampering hasn't occurred (SHA-256 digest files)
- **IAM is global** — use a multi-region trail (or global services events enabled) to capture IAM API calls
- **CloudTrail Lake** = no S3/Athena setup needed; pay per GB ingested and scanned
- **Insights Events** require existing Management Events to be enabled first
- **Root account activity** — CloudTrail always logs root usage; monitor it via CloudWatch Metric Filter + Alarm
- CloudTrail logs are delivered **within 15 minutes** of API call — not real-time (use EventBridge for real-time)
- **SSE-KMS encryption** gives you a key usage audit trail — the CMK usage itself appears in CloudTrail

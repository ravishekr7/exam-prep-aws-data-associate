# AWS CloudTrail

Auditing service that records WHO did WHAT API calls in your AWS account.

---

## Event Types

| Type | Description | Default |
|------|-------------|---------|
| **Management Events** | Control plane operations (CreateBucket, RunJob, CreateUser) | Enabled by default |
| **Data Events** | Data plane operations (S3 GetObject/PutObject, Lambda Invoke) | NOT enabled by default (costly) |
| **Insights Events** | Unusual API activity detection | Optional |

---

## CloudTrail Lake

- Centralized querying of CloudTrail events using SQL
- Replace the need to export logs to S3 + Athena
- Managed storage and query engine
- Retention: up to 7 years

---

## Key Use Cases

- **"Who deleted this S3 object?"** = Enable Data Events on S3
- **"What API calls were made?"** = CloudTrail Management Events
- **"Audit trail for compliance"** = CloudTrail -> S3 -> Athena (or CloudTrail Lake)
- **"Unusual activity detection"** = CloudTrail Insights

---

## Exam Gotchas

- Data Events are **NOT enabled by default** (expensive) - must explicitly enable for S3 object-level auditing
- Management Events are free and enabled by default
- CloudTrail Lake = SQL queries on CloudTrail events without setting up S3 + Athena
- CloudTrail is for **API auditing**, CloudWatch is for **performance monitoring**
- Logs can be sent to S3 and analyzed with Athena for cost-effective historical queries

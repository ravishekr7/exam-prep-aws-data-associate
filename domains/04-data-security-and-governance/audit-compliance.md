# Audit and Compliance

Logging, auditing, and compliance patterns for data engineering.

---

## Logging Services Comparison

| Service | Tracks | Use Case |
|---------|--------|----------|
| **CloudTrail** | API calls (who did what) | Security audit, compliance |
| **CloudWatch Logs** | Application logs, metrics | Operational monitoring |
| **AWS Config** | Resource configuration changes | Compliance, drift detection |
| **VPC Flow Logs** | Network traffic metadata | Network troubleshooting, security |

---

## Audit Architecture

### Standard Pattern
```
CloudTrail -> S3 -> Athena (query with SQL)
```

### Advanced Pattern (CloudTrail Lake)
```
CloudTrail -> CloudTrail Lake -> SQL queries directly
```
- No need for S3 + Athena setup
- Managed retention up to 7 years
- Simpler for organizations that primarily need CloudTrail analysis

### Log Analysis at Scale
```
Application Logs -> CloudWatch Logs -> S3 -> Athena/EMR/OpenSearch
```
- OpenSearch for real-time log analysis and visualization (Kibana dashboards)
- Athena for ad-hoc SQL queries on historical logs
- EMR for large-volume log processing

---

## Compliance Patterns

### Encryption Compliance
- Enable default encryption on all S3 buckets (SSE-S3 or SSE-KMS)
- Use AWS Config rules to detect non-compliant buckets
- Enforce encryption in transit with bucket policies (`aws:SecureTransport`)

### Access Compliance
- CloudTrail Data Events for S3 object-level audit trail
- Lake Formation for fine-grained data access governance
- IAM Access Analyzer to identify resources shared externally

### Data Retention Compliance
- S3 Object Lock for WORM (Write Once Read Many)
- Glacier Vault Lock for immutable archive policies
- S3 Lifecycle policies for automated data retention/deletion

---

## Exam Gotchas

- **"Audit API calls"** = CloudTrail
- **"Audit configuration changes"** = AWS Config
- **"Application logs"** = CloudWatch Logs
- **"Query audit logs with SQL"** = CloudTrail -> S3 -> Athena (or CloudTrail Lake)
- **"Real-time log dashboard"** = CloudWatch Logs -> OpenSearch + Kibana
- **"WORM compliance"** = S3 Object Lock or Glacier Vault Lock
- CloudTrail Lake simplifies audit querying without S3/Athena setup

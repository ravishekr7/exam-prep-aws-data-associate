# Data Privacy and PII

Protecting sensitive data and meeting compliance requirements.

---

## PII Detection

### Amazon Macie
- Scans **S3 buckets** using ML to find sensitive data
- Detects: Credit card numbers, SSNs, names, addresses, API keys
- Integrates with Lake Formation for automated response
- Generates findings and alerts via EventBridge

### AWS Glue PII Detection
- **FindMatches ML Transform:** Identify duplicate/similar records
- **PII Detection Transform:** Detect PII in Glue ETL jobs
- Can **mask** PII (replace with `***`) or **hash** it
- Runs inline during ETL processing

---

## Data Masking Patterns

### Masking in Transit (Before Landing)
```
Source -> Kinesis Firehose -> Lambda (mask PII) -> S3
```
- Raw data **never touches disk** - masked before S3 delivery
- Use when compliance says unmasked PII must never be stored

### Masking During Processing
```
S3 (raw) -> Glue Job (PII transform) -> S3 (masked)
```
- Process existing data and create masked copies
- Raw data exists temporarily in S3

---

## AWS Config

- Tracks **configuration changes** to AWS resources over time
- "Was this S3 bucket public yesterday?" -> Config tells you
- **Config Rules:** Evaluate compliance (e.g., "All S3 buckets must have encryption enabled")
- **Conformance Packs:** Pre-built rule sets for frameworks (HIPAA, PCI-DSS)

---

## Data Sovereignty

- Keep data within specific geographic regions
- Use S3 bucket region restrictions
- Prevent cross-region replication to disallowed regions
- SCP (Service Control Policies) can restrict which regions services can be used in

---

## Data Sharing and Governance

### Redshift Data Sharing
- Share live Redshift data across clusters/accounts without copying
- Producer shares a datashare, consumer creates a database from it
- Cross-region supported

### AWS Data Exchange
- Find and subscribe to third-party data
- Deliver data to S3
- Marketplace for data products

---

## Exam Gotchas

- **"Find PII in S3"** = Macie
- **"Mask PII during ETL"** = Glue PII Detection transform
- **"PII never touches disk"** = Firehose + Lambda transformation (mask before S3)
- **"Track configuration changes"** = AWS Config
- **"Track API calls"** = CloudTrail (different from Config)
- **"Prevent resources in certain regions"** = SCP (Service Control Policies)
- Macie is for S3 only; Glue handles PII detection in ETL pipelines

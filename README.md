# AWS Certified Data Engineer - Associate (DEA-C01)

Preparation repository for the AWS Certified Data Engineer - Associate exam.

**Exam Details:**
- Code: DEA-C01
- Duration: 130 minutes | 65 questions (50 scored + 15 unscored)
- Passing Score: 720/1000
- Exam Date: **April 22, 2026**
- Cost: $150 USD
- Prerequisites: 2-3 years data engineering + 1-2 years hands-on AWS

---

## Exam Domains

| # | Domain | Weight | Folder |
|---|--------|--------|--------|
| 1 | Data Ingestion and Transformation | **34%** | [domains/01-data-ingestion-and-transformation](./domains/01-data-ingestion-and-transformation/) |
| 2 | Data Store Management | **26%** | [domains/02-data-store-management](./domains/02-data-store-management/) |
| 3 | Data Operations and Support | **22%** | [domains/03-data-operations-and-support](./domains/03-data-operations-and-support/) |
| 4 | Data Security and Governance | **18%** | [domains/04-data-security-and-governance](./domains/04-data-security-and-governance/) |

## Additional Resources

| Section | Description |
|---------|-------------|
| [scenarios/](./scenarios/) | Architecture patterns, decision matrices, and advanced exam scenarios |
| [practice/](./practice/) | Exam tips, question patterns, and gotchas |

---

## 3-Layer Study Method

Use this daily framework to build deep, durable knowledge — not surface-level recall.

```
Daily Session (approx. 110 minutes):

1. UDEMY (30 min)
   → Watch the relevant lecture for today's topic
   → Build context: WHY this service exists, WHAT problem it solves
   → Don't take notes yet — just absorb the narrative

2. REPO NOTES (45 min)
   → Read the relevant .md file(s) for today's topic
   → This is where depth lives: formulas, gotchas, comparison tables, code snippets
   → Highlight anything surprising or counter-intuitive — those are exam traps

3. AWS DOCS (20 min)
   → Pick ONE specific topic from today's file that felt fuzzy
   → Read the official AWS documentation for ground truth
   → Official docs contain the authoritative limits, default values, and constraints

4. DAILY QUIZ (15 min)
   → Answer 3-5 questions from practice/question-patterns.md for today's domain
   → If you got it wrong: re-read the relevant section and annotate why you missed it
   → Track weak areas — these become your Week 4 focus
```

**Why this works:** Udemy gives you the mental model. Repo notes give you the exam-specific precision. AWS docs prevent you from memorizing outdated or incorrect information. Daily quizzes force active recall, which is 2-3× more effective than re-reading.

---

## Key Numbers to Memorise

These specific numbers appear on the exam. If you have to calculate, start from these.

| Service | Key Limit / Formula | Notes |
|---------|--------------------|----|
| **Kinesis Data Streams** | Write: 1 MB/s or 1,000 rec/s per shard | Both limits apply — shard count = max of all constraints |
| **Kinesis Data Streams** | Read: 2 MB/s per shard (shared) | EFO: 2 MB/s DEDICATED per consumer per shard |
| **Kinesis Firehose** | Minimum buffer: 60 seconds | NOT suitable for sub-minute delivery SLAs |
| **Kinesis Firehose** | Maximum buffer: 900 seconds (15 min) | Buffer size: 1–128 MB |
| **Lambda** | Max timeout: 15 minutes | Max memory: 10,240 MB (10 GB) |
| **Lambda** | Max deployment package: 250 MB (unzipped) | Max concurrent executions: 1,000 (default, can increase) |
| **DynamoDB RCU** | Strong: ceil(item_KB / 4) | Eventual: half of strong (divide by 2) |
| **DynamoDB WCU** | ceil(item_KB / 1) | Transactional: 2× the standard RCU or WCU |
| **DynamoDB** | Max item size: 400 KB | LSI: must create at table creation time |
| **DynamoDB TTL** | Expires within 48 hours of timestamp | No WCU cost |
| **S3 Standard** | Min storage duration: none | Frequent access tier |
| **S3 Standard-IA** | Min storage duration: 30 days | Minimum 128 KB object size billing |
| **S3 Glacier Instant** | Min storage duration: 90 days | Millisecond retrieval |
| **S3 Glacier Flexible** | Min storage duration: 90 days | Minutes to hours retrieval |
| **S3 Glacier Deep Archive** | Min storage duration: 180 days | 12+ hours retrieval |
| **SQS** | Max message retention: 14 days | Max message size: 256 KB |
| **SQS** | Default visibility timeout: 30 seconds | Must be >= Lambda function timeout |
| **Step Functions Standard** | Max duration: 1 year | Exactly-once semantics |
| **Step Functions Express** | Max duration: 5 minutes | At-least-once semantics |
| **Redshift** | Default shuffle partitions (Spark): 200 | Target: ~128 MB per partition |
| **Glue** | Cold start: 10–30 seconds | Min billing: 1 DPU-minute |
| **DynamoDB Streams** | Retention: 24 hours | Records after: INSERT, MODIFY, REMOVE |

---

## High-Frequency Exam Topics

Based on community experience and exam guide weightings, these 10 services appear most frequently in DEA-C01 questions. Allocate study time proportionally.

| Rank | Service | Why It's High-Frequency |
|------|---------|------------------------|
| 1 | **AWS Glue** | ETL, Data Catalog, Schema Registry, Data Quality — central to Domain 1 |
| 2 | **Amazon Kinesis** | KDS + Firehose + Flink — streaming is 34% of the exam (Domain 1) |
| 3 | **Amazon S3** | Storage foundation for every scenario — partitioning, lifecycle, encryption, table formats |
| 4 | **Amazon Redshift** | Distribution/sort keys, WLM, Spectrum — heavy Domain 2 content |
| 5 | **AWS Lake Formation** | Column/row/tag security — entire Domain 4 layer for data lakes |
| 6 | **Amazon Athena** | Serverless SQL on S3 — appears in analytics, cost, and architecture questions |
| 7 | **Amazon DynamoDB** | RCU/WCU math, GSI vs LSI, DAX, Streams — multiple question types |
| 8 | **AWS Lambda** | Event processing, error handling, timeouts — appears in every domain |
| 9 | **AWS Step Functions** | Retry/Catch syntax, Standard vs Express, orchestration — Domain 3 |
| 10 | **AWS KMS** | SSE modes, CMK vs AWS-managed, CloudTrail audit — Domain 4 |

**Study tip:** For each top-10 service, you should be able to answer: "When would I choose this over its alternatives?" and "What are the key limits and gotchas?"

---

## 30-Day Study Plan

### Week 1: Domain 1 - Data Ingestion and Transformation (34%)
*Highest weighted domain — invest the most time here.*

| Day | Topic | File |
|-----|-------|------|
| 1 | AWS Glue (Catalog, Crawlers, ETL Jobs, Streaming ETL, DynamicFrame, Connections) | [aws-glue.md](./domains/01-data-ingestion-and-transformation/aws-glue.md) |
| 2 | Amazon Kinesis (KDS, Shard Math, EFO, Firehose, Flink) | [amazon-kinesis.md](./domains/01-data-ingestion-and-transformation/amazon-kinesis.md) |
| 3 | AWS DMS, Schema Conversion Tool, CDC | [aws-dms.md](./domains/01-data-ingestion-and-transformation/aws-dms.md) |
| 4 | S3 Ingestion Patterns + Amazon AppFlow | [amazon-s3-ingestion.md](./domains/01-data-ingestion-and-transformation/amazon-s3-ingestion.md) |
| 5 | Amazon EMR (Deployment Modes, Spark Tuning, Instance Fleets) + Amazon MSK | [amazon-emr.md](./domains/01-data-ingestion-and-transformation/amazon-emr.md) / [amazon-msk.md](./domains/01-data-ingestion-and-transformation/amazon-msk.md) |
| 6 | Orchestration: Step Functions + MWAA (Environment Sizing, XCom vs S3) | [aws-step-functions.md](./domains/01-data-ingestion-and-transformation/aws-step-functions.md) / [amazon-mwaa.md](./domains/01-data-ingestion-and-transformation/amazon-mwaa.md) |
| 7 | Lambda, Programming Concepts, IaC, CI/CD | [aws-lambda.md](./domains/01-data-ingestion-and-transformation/aws-lambda.md) / [programming-concepts.md](./domains/01-data-ingestion-and-transformation/programming-concepts.md) |

### Week 2: Domain 2 - Data Store Management (26%)

| Day | Topic | File |
|-----|-------|------|
| 8 | Amazon S3 Data Lake (Storage Classes, Partitioning, Lifecycle) | [amazon-s3-data-lake.md](./domains/02-data-store-management/amazon-s3-data-lake.md) |
| 9 | Amazon Redshift (Distribution/Sort Key Decision Guides, WLM, Debugging) | [amazon-redshift.md](./domains/02-data-store-management/amazon-redshift.md) |
| 10 | Amazon DynamoDB (RCU/WCU Math, GSI vs LSI, Streams+Lambda, Export to S3) | [amazon-dynamodb.md](./domains/02-data-store-management/amazon-dynamodb.md) |
| 11 | Amazon RDS, Aurora, DocumentDB, Neptune, Keyspaces | [amazon-rds-aurora.md](./domains/02-data-store-management/amazon-rds-aurora.md) |
| 12 | AWS Lake Formation + Glue Data Catalog | [aws-lake-formation.md](./domains/02-data-store-management/aws-lake-formation.md) |
| 13 | Data Modeling (Star/Snowflake, OLTP vs OLAP) + File Formats | [data-modeling.md](./domains/02-data-store-management/data-modeling.md) / [file-formats-compression.md](./domains/02-data-store-management/file-formats-compression.md) |
| 14 | Open Table Formats (Iceberg deep dive, Hudi CoW vs MoR, Delta, S3 Tables) | [open-table-formats.md](./domains/02-data-store-management/open-table-formats.md) |

### Week 3: Domain 3 + Domain 4 (22% + 18%)

| Day | Topic | File |
|-----|-------|------|
| 15 | Orchestration Deep Dive (Step Functions Standard vs Express, MWAA) | [orchestration.md](./domains/03-data-operations-and-support/orchestration.md) |
| 16 | Monitoring: CloudWatch Metrics/Logs/Alarms (per-service metrics) | [monitoring-cloudwatch.md](./domains/03-data-operations-and-support/monitoring-cloudwatch.md) |
| 17 | CloudTrail + Athena for Log Analysis | [aws-cloudtrail.md](./domains/03-data-operations-and-support/aws-cloudtrail.md) / [amazon-athena.md](./domains/03-data-operations-and-support/amazon-athena.md) |
| 18 | Data Quality (Glue DQ DQDL deep dive, actions) + Error Handling (DLQ, Kinesis patterns) | [data-quality.md](./domains/03-data-operations-and-support/data-quality.md) / [error-handling.md](./domains/03-data-operations-and-support/error-handling.md) |
| 19 | Cost Optimization + Performance Tuning (Spark, Redshift, Kinesis) | [cost-optimization.md](./domains/03-data-operations-and-support/cost-optimization.md) / [performance-tuning.md](./domains/03-data-operations-and-support/performance-tuning.md) |
| 20 | IAM (Policy Evaluation Logic, Permission Boundaries, Cross-Account) | [iam.md](./domains/04-data-security-and-governance/iam.md) |
| 21 | Lake Formation Security (Permission Model, LF-Tags, Row/Column Security) | [lake-formation-security.md](./domains/04-data-security-and-governance/lake-formation-security.md) |
| 22 | Encryption (KMS, SSE modes, Envelope Encryption) | [encryption-kms.md](./domains/04-data-security-and-governance/encryption-kms.md) |
| 23 | Network Security (VPC Endpoint Types, Endpoint Policies, PrivateLink for data services) + PII/Macie | [network-security.md](./domains/04-data-security-and-governance/network-security.md) / [data-privacy-pii.md](./domains/04-data-security-and-governance/data-privacy-pii.md) |

### Week 4: Scenarios, Practice, and Review

| Day | Topic | File |
|-----|-------|------|
| 24 | Decision Matrices (all 6 comparison tables) | [decision-matrices.md](./scenarios/decision-matrices.md) |
| 25 | Architecture Patterns (Lakehouse, Real-Time Analytics, CDC) | [architecture-patterns.md](./scenarios/architecture-patterns.md) |
| 26 | Advanced Scenarios (Fraud Detection, GDPR, Cross-Account, CDC Pipeline) | [advanced-scenarios.md](./scenarios/advanced-scenarios.md) |
| 27 | 25 Practice Questions — Score yourself, annotate missed questions | [question-patterns.md](./practice/question-patterns.md) |
| 28 | Exam Tips + Anti-Patterns + Weak Area Review | [exam-tips.md](./practice/exam-tips.md) |
| 29 | Practice Questions — ExamTopics Set 1 + review notes on missed questions | External |
| 30 | Full Final Review — Key Numbers table, Exam Gotchas from all domain files | This file + all domain README files |

---

## In-Scope AWS Services (Complete List)

### Analytics
Amazon Athena, Amazon EMR, AWS Glue, AWS Glue DataBrew, AWS Lake Formation, Amazon Kinesis Data Firehose, Amazon Kinesis Data Streams, Amazon Managed Service for Apache Flink, Amazon MSK, Amazon OpenSearch Service, Amazon QuickSight, Amazon SageMaker AI

### Application Integration
Amazon AppFlow, Amazon EventBridge, Amazon MWAA, Amazon SNS, Amazon SQS, AWS Step Functions

### Compute
AWS Batch, Amazon EC2, AWS Lambda, AWS SAM

### Containers
Amazon ECR, Amazon ECS, Amazon EKS

### Database
Amazon DocumentDB, Amazon DynamoDB, Amazon Keyspaces, Amazon MemoryDB, Amazon Neptune, Amazon RDS, Amazon Aurora, Amazon Redshift

### Developer Tools
AWS CLI, AWS CloudFormation, AWS CDK, AWS CodeBuild, AWS CodeDeploy, AWS CodePipeline, Amazon Q

### Machine Learning
Amazon SageMaker AI, Amazon Bedrock, Amazon Kendra

### Management and Governance
AWS CloudTrail, Amazon CloudWatch, CloudWatch Logs, AWS Config, Amazon Managed Grafana, AWS Systems Manager, AWS Well-Architected Tool, AWS Data Exchange

### Migration and Transfer
AWS Application Discovery Service, AWS Application Migration Service, AWS DMS, AWS DataSync, AWS Snow Family, AWS Transfer Family

### Networking
Amazon CloudFront, AWS PrivateLink, Amazon Route 53, Amazon VPC

### Security
IAM, AWS KMS, Amazon Macie, AWS Secrets Manager, AWS Shield, AWS WAF

### Storage
AWS Backup, Amazon EBS, Amazon EFS, Amazon S3, Amazon S3 Tables, Amazon S3 Glacier

### Web and Mobile
Amazon API Gateway

---

## Quick Links

- [Official Exam Guide](https://docs.aws.amazon.com/aws-certification/latest/data-engineer-associate-01/data-engineer-associate-01.html)
- [Exam Registration](https://aws.amazon.com/certification/certified-data-engineer-associate/)
- [Practice Questions (ExamTopics)](https://www.examtopics.com/exams/amazon/aws-certified-data-engineer-associate-dea-c01/view/)

# AWS Certified Data Engineer - Associate (DEA-C01)

Preparation repository for the AWS Certified Data Engineer - Associate exam.

**Exam Details:**
- Code: DEA-C01
- Duration: 130 minutes | 65 questions (50 scored + 15 unscored)
- Passing Score: 720/1000
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

## 30-Day Study Plan

### Week 1: Domain 1 - Data Ingestion and Transformation (34%)
*Highest weighted domain - invest the most time here.*

| Day | Topic | File |
|-----|-------|------|
| 1 | AWS Glue (Catalog, Crawlers, ETL Jobs, DynamicFrame) | [aws-glue.md](./domains/01-data-ingestion-and-transformation/aws-glue.md) |
| 2 | Amazon Kinesis (KDS, Firehose, Flink) | [amazon-kinesis.md](./domains/01-data-ingestion-and-transformation/amazon-kinesis.md) |
| 3 | AWS DMS, Schema Conversion Tool, CDC | [aws-dms.md](./domains/01-data-ingestion-and-transformation/aws-dms.md) |
| 4 | S3 Ingestion Patterns + Amazon AppFlow | [amazon-s3-ingestion.md](./domains/01-data-ingestion-and-transformation/amazon-s3-ingestion.md) |
| 5 | Amazon EMR + Amazon MSK | [amazon-emr.md](./domains/01-data-ingestion-and-transformation/amazon-emr.md) / [amazon-msk.md](./domains/01-data-ingestion-and-transformation/amazon-msk.md) |
| 6 | Orchestration: Step Functions + MWAA | [aws-step-functions.md](./domains/01-data-ingestion-and-transformation/aws-step-functions.md) / [amazon-mwaa.md](./domains/01-data-ingestion-and-transformation/amazon-mwaa.md) |
| 7 | Lambda, Programming Concepts, IaC, CI/CD | [aws-lambda.md](./domains/01-data-ingestion-and-transformation/aws-lambda.md) / [programming-concepts.md](./domains/01-data-ingestion-and-transformation/programming-concepts.md) |

### Week 2: Domain 2 - Data Store Management (26%)

| Day | Topic | File |
|-----|-------|------|
| 8 | Amazon S3 Data Lake (Storage Classes, Partitioning, Lifecycle) | [amazon-s3-data-lake.md](./domains/02-data-store-management/amazon-s3-data-lake.md) |
| 9 | Amazon Redshift (Architecture, Spectrum, Distribution/Sort Keys) | [amazon-redshift.md](./domains/02-data-store-management/amazon-redshift.md) |
| 10 | Amazon DynamoDB (Keys, GSI/LSI, DAX, Capacity Modes) | [amazon-dynamodb.md](./domains/02-data-store-management/amazon-dynamodb.md) |
| 11 | Amazon RDS, Aurora, DocumentDB, Neptune, Keyspaces | [amazon-rds-aurora.md](./domains/02-data-store-management/amazon-rds-aurora.md) |
| 12 | AWS Lake Formation + Glue Data Catalog | [aws-lake-formation.md](./domains/02-data-store-management/aws-lake-formation.md) |
| 13 | Data Modeling (Star/Snowflake, OLTP vs OLAP) + File Formats | [data-modeling.md](./domains/02-data-store-management/data-modeling.md) / [file-formats-compression.md](./domains/02-data-store-management/file-formats-compression.md) |
| 14 | Open Table Formats (Iceberg, Hudi, Delta) + Vector Concepts | [open-table-formats.md](./domains/02-data-store-management/open-table-formats.md) |

### Week 3: Domain 3 + Domain 4 (22% + 18%)

| Day | Topic | File |
|-----|-------|------|
| 15 | Orchestration Deep Dive (Step Functions, MWAA) | [orchestration.md](./domains/03-data-operations-and-support/orchestration.md) |
| 16 | Monitoring: CloudWatch Metrics/Logs/Alarms | [monitoring-cloudwatch.md](./domains/03-data-operations-and-support/monitoring-cloudwatch.md) |
| 17 | CloudTrail + Athena for Log Analysis | [aws-cloudtrail.md](./domains/03-data-operations-and-support/aws-cloudtrail.md) / [amazon-athena.md](./domains/03-data-operations-and-support/amazon-athena.md) |
| 18 | Data Quality (Glue DQ, DataBrew) + Error Handling (DLQ, Retries) | [data-quality.md](./domains/03-data-operations-and-support/data-quality.md) / [error-handling.md](./domains/03-data-operations-and-support/error-handling.md) |
| 19 | Cost Optimization + Performance Tuning (Spark, Redshift, Kinesis) | [cost-optimization.md](./domains/03-data-operations-and-support/cost-optimization.md) / [performance-tuning.md](./domains/03-data-operations-and-support/performance-tuning.md) |
| 20 | IAM (Roles, Policies, Cross-Account) | [iam.md](./domains/04-data-security-and-governance/iam.md) |
| 21 | Lake Formation Security (Row/Column/Tag-based Access) | [lake-formation-security.md](./domains/04-data-security-and-governance/lake-formation-security.md) |
| 22 | Encryption (KMS, SSE modes, Envelope Encryption) | [encryption-kms.md](./domains/04-data-security-and-governance/encryption-kms.md) |
| 23 | Network Security (VPC Endpoints, SGs, NACLs) + PII/Macie | [network-security.md](./domains/04-data-security-and-governance/network-security.md) / [data-privacy-pii.md](./domains/04-data-security-and-governance/data-privacy-pii.md) |

### Week 4: Scenarios, Practice, and Review

| Day | Topic | File |
|-----|-------|------|
| 24 | Decision Matrices (Kinesis vs MSK vs SQS, Redshift vs Athena vs EMR) | [decision-matrices.md](./scenarios/decision-matrices.md) |
| 25 | Architecture Patterns (Lakehouse, Real-Time Analytics, CDC) | [architecture-patterns.md](./scenarios/architecture-patterns.md) |
| 26 | Advanced Scenarios (Fraud Detection, GDPR, PII Masking, Cross-Account) | [advanced-scenarios.md](./scenarios/advanced-scenarios.md) |
| 27 | Exam Tips + Question Patterns + Anti-Patterns | [exam-tips.md](./practice/exam-tips.md) |
| 28 | Practice Questions - ExamTopics Set 1 | Review + take notes |
| 29 | Practice Questions - ExamTopics Set 2 | Review weak areas |
| 30 | Full Review - Revisit all domain READMEs and gotchas | Final pass |

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

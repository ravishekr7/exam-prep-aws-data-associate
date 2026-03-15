# Exam Tips and Question Patterns

Strategies for answering DEA-C01 exam questions.

---

## Keyword-to-Answer Mapping

When you see these keywords in a question, lean toward these answers:

| Keyword | Answer |
|---------|--------|
| "Highly available" | Multi-AZ |
| "Global" / "Multi-region" | DynamoDB Global Tables / Aurora Global Database |
| "Serverless SQL" | Athena |
| "Complex joins" / "BI reporting" | Redshift |
| "Schema evolution" | Glue Schema Registry / Avro |
| "Sub-second" / "millisecond latency" | DynamoDB / DAX |
| "Near real-time delivery to S3" | Kinesis Firehose |
| "Real-time processing" (sub-second) | Flink (Kinesis Data Analytics) |
| "Existing Kafka" / "Kafka migration" | MSK |
| "SaaS data source" | AppFlow |
| "Most cost-effective storage" | S3 Intelligent-Tiering or Lifecycle policies |
| "Least privilege" | Most restrictive IAM policy that works |
| "Column-level security" | Lake Formation |
| "PII detection in S3" | Macie |
| "Audit API calls" | CloudTrail |
| "Configuration compliance" | AWS Config |
| "Orchestrate multiple services" | Step Functions |
| "Backfilling" / "existing Airflow" | MWAA |
| "Row-level updates on S3" | Apache Iceberg |
| "Cross-account encrypted data" | Customer Managed KMS Key |
| "Minimize data scanned" | Parquet + Partitioning |
| "Handle failed messages" | Dead Letter Queue |
| "Handle throttling" | Exponential backoff |

---

## Elimination Strategy

### Step 1: Identify the Constraint
Every scenario has a key constraint. Find it first:
- **Latency:** Real-time vs near-real-time vs batch
- **Cost:** "Most cost-effective" or "minimize cost"
- **Management:** "Least operational overhead" or "fully managed"
- **Compliance:** "Must never touch disk unencrypted" or "audit trail required"

### Step 2: Eliminate Wrong Services
- Lambda mentioned for > 15 min processing? WRONG
- RDS for analytics? WRONG
- Firehose for sub-second latency? WRONG (60s minimum)
- Glue for sub-second response? WRONG (cold start)

### Step 3: Among Remaining, Pick the Simplest
AWS exams favor managed/serverless solutions that solve the problem with least overhead.

---

## Common Traps

1. **"Migrate existing Kafka"** - Answer is MSK, not Kinesis (even though Kinesis can do similar things)
2. **"Serverless"** does NOT mean cheapest. Provisioned can be cheaper at steady high load
3. **Multi-AZ is for HA**, Read Replicas are for read scaling (different purposes)
4. **S3 Gateway Endpoint is free**, Interface Endpoint costs money
5. **CloudTrail Data Events are NOT default** - you must enable them for S3 object-level auditing
6. **SSE-KMS has API quotas** - high throughput may need S3 Bucket Keys
7. **Glue Crawlers cost money** - don't use them when schema is known and static

---

## Question Pattern: "Which combination of steps?"

These questions ask for multiple correct steps. Strategy:
1. Identify the end goal
2. Work backwards from the goal to determine required components
3. Eliminate any step that doesn't contribute to the goal
4. Check that all steps work together (e.g., permissions, network, format compatibility)

---

## Time Management

- 130 minutes for 65 questions = ~2 minutes per question
- Flag difficult questions and come back
- Read ALL answer choices before selecting
- "Most cost-effective" questions usually have one clearly cheaper option
- "Least operational overhead" usually means serverless/managed

---

## Day Before Exam

1. Review all domain READMEs for high-level coverage
2. Re-read the decision matrices (scenarios/decision-matrices.md)
3. Re-read the advanced scenarios (scenarios/advanced-scenarios.md)
4. Focus on your weakest domain
5. Get good sleep - the exam is 2+ hours of concentration
